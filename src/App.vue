<template>
  <div class="p-4 flex flex-col items-center">
    <h1 class="text-2xl font-bold mb-4 text-center">VOICEVOX BPM SYNC</h1>

    <label class="label">モード</label>
    <select class="select select-bordered w-full max-w-xs" v-model="mode">
      <option>プロジェクトファイルを変換</option>
      <option>音声を合成</option>
    </select>
    
    <div v-if="mode === '音声を合成'" class="flex flex-col items-center">

      <label class="label">BPM</label>
      <input type="number" v-model="bpm" class="input input-bordered w-full max-w-xs" />

      <label class="label">話者</label>
      <select v-model="selectedSpeakerUuid" class="select select-bordered w-full max-w-xs" @change="selectedStyleIndex = 0">
        <option v-for="speaker in speakers" :key="speaker.speaker_uuid" :value="speaker.speaker_uuid">{{ speaker.name }}</option>
      </select>

      <label class="label">スタイル</label>
      <select v-model="selectedStyleIndex" class="select select-bordered w-full max-w-xs">
        <option v-for="(style, index) in selectedSpeakerStyles" :key="style.id" :value="index">{{ style.name }}</option>
      </select>

      <label class="label">テキスト</label>
      <textarea v-model="content" class="textarea textarea-bordered w-full max-w-xs" rows="10"></textarea>

      <button @click="generate" class="btn btn-primary mt-4 mb-4 w-full max-w-xs">音声合成</button>

      <audio id="audio" controls></audio>
    </div>
    
    <div v-else class="flex flex-col items-center">

      <label class="label">BPM</label>
      <input type="number" v-model="bpm" class="input input-bordered w-full max-w-xs" />

      <label class="label">プロジェクトファイル</label>
      <input type="file" class="file-input file-input-bordered w-full max-w-xs" @change="display" />
      
      <!-- 音価設定用の表示を追加 -->
      <div v-for="(queries, audioId) in queriesByAudioItems" :key="audioId" class="my-4 w-full">
        <div>{{ queries[0].text }}</div>
        <label class="label">音価一括変更</label>
        <select class="select select-bordered w-full max-w-xs" @change="updateNoteValue(audioId, $event)">
          <option v-for="option in noteOptions" :key="option.value" :value="option.value">{{ option.label }}</option>
        </select>
        <div class="flex flex-wrap" v-for="query in queries" :key="query.id">
          <div v-for="mora in query.moras" :key="mora.id" class="flex flex-col items-center my-2">
            <span>{{ mora.text }}</span>
            <select v-model="mora.noteValue" class="select select-bordered ml-2">
              <option v-for="option in noteOptions" :key="option.value" :value="option.value">{{ option.label }}</option>
            </select>
          </div>
        </div>
      </div>
      <!-- 変換を実行するボタンを追加 -->
      <button @click="convert" class="btn btn-primary mt-4 mb-4 w-full max-w-xs">プロジェクトファイルを変換</button>
      
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue'

interface Mora {
  text: string;
  consonantLength?: number | null;
  vowelLength?: number | null;
}

interface AccentPhrase {
  moras: Mora[];
  pauseMora?: Mora;
}

interface AudioItem {
  text: string;
  query: {
    accentPhrases: AccentPhrase[];
  };
}

interface Data {
  talk:{
    audioItems: { [key: string]: AudioItem };
  }
}

interface DisplayMora {
  id: number;
  text: string;
  noteValue: number;
}

interface Query {
  id: number;
  text: string;
  moras: DisplayMora[];
}

export default defineComponent({
  name: 'App',
  data() {
    return {
      bpm: 120,
      speakers: [] as any,
      selectedSpeakerUuid: '',
      selectedStyleIndex: 0,
      content: '',
      query: {} as any,
      mode: 'プロジェクトファイルを変換',
      queriesByAudioItems: {} as { [key: string]: Query[] },
      noteOptions: [
        { value: 2, label: '1/4' },
        { value: 3, label: '1/4T' },
        { value: 4, label: '1/8' },
        { value: 6, label: '1/8T' },
        { value: 8, label: '1/16' },
        { value: 12, label: '1/16T' },
        { value: 16, label: '1/32' },
      ],
      data: null as Data | null,
      fileName: '',
    }
  },
  computed: {
    selectedSpeaker() {
      return this.speakers.find((speaker: any) => speaker.speaker_uuid === this.selectedSpeakerUuid)
    },
    selectedSpeakerStyles() {
      return this.selectedSpeaker ? this.selectedSpeaker.styles : []
    },
    selectedStyle() {
      return this.selectedSpeakerStyles[this.selectedStyleIndex]
    },
  },
  mounted() {
    fetch('http://localhost:50021/speakers')
      .then((res) => res.json())
      .then((data) => {
        this.speakers = data;
        this.selectedSpeakerUuid = data[0].speaker_uuid;
      });
  },
  methods: {
    generate() {
      const queryParams = new URLSearchParams({
        text: this.content,
        speaker: this.selectedStyle.id,
      }).toString();

      fetch(`http://localhost:50021/audio_query?${queryParams}`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({}),
      })
        .then((res) => res.json())
        .then((data) => {
          this.query = data;
          for (let accent_phrase of this.query.accent_phrases) {
            const updateMora = (mora: any) => {
              let sum = (mora.consonant_length || 0) + (mora.vowel_length || 0);
              let rhythm = 4;
              let rate = 60 / this.bpm / rhythm / sum;

              if (mora.consonant_length) mora.consonant_length *= rate;
              if (mora.vowel_length) mora.vowel_length *= rate;
            }
            for (let mora of accent_phrase.moras) {          
              updateMora(mora);
            }

            if (accent_phrase.pause_mora) {
              updateMora(accent_phrase.pause_mora);
            }
          }

          const queryParams = new URLSearchParams({
            speaker: this.selectedStyle.id,
          }).toString();

          fetch(`http://localhost:50021/synthesis?${queryParams}`, {
            method: 'POST',
            headers: {
              'Content-Type': 'application/json',
            },
            body: JSON.stringify(this.query),
          })
            .then((res) => res.blob())
            .then((blob) => {
              const audio = document.getElementById('audio') as HTMLAudioElement;
              audio.src = URL.createObjectURL(blob);
              audio.play();
            })
        })
    },
    convert() {

      if (!this.data) return;
      
      const bpm = this.bpm;

      // console.log('Data loaded:', this.data);

      const getNoteValue = (audioItemId: string, moraIndex: number): number => {
        const queries = this.queriesByAudioItems[audioItemId];
        if (queries) {
          for (const query of queries) {
            if (query.moras[moraIndex]) {
              return query.moras[moraIndex].noteValue;
            }
          }
        }
        return 8;
      };

      for (let audioItemId in this.data.talk.audioItems) {
        const audioItem = this.data.talk.audioItems[audioItemId];

        let moraIndex = 0;

        for (let accentPhrase of audioItem.query.accentPhrases) {
          const updateMora = (mora: Mora, moraIndex: number) => {
            const noteValue = getNoteValue(audioItemId, moraIndex);
            // console.log('Get noteValue:', noteValue);
            const noteLength = 60 / bpm / noteValue;
            const totalLength = (mora.consonantLength || 0) + (mora.vowelLength || 0);

            if (mora.consonantLength) {
              const consonantRatio = mora.consonantLength / totalLength;
              mora.consonantLength = noteLength * consonantRatio;
              mora.vowelLength = noteLength * (1 - consonantRatio);
            } else {
              mora.vowelLength = noteLength;
            }
          };
          for (let mora of accentPhrase.moras) {
            // console.log(`Original mora:`, mora);
            updateMora(mora, moraIndex++);
            // console.log(`Updated mora:`, mora);
          }

          if (accentPhrase.pauseMora) {
            updateMora(accentPhrase.pauseMora, moraIndex++);
          }
        }
      }

      const blob = new Blob([JSON.stringify(this.data)], { type: "text/plain" });
      const link = document.createElement("a");
      link.href = URL.createObjectURL(blob);
      link.download = this.fileName;
      link.click();
    },
    display(event: any){
      const file = event.target.files[0];
      this.fileName = file.name;
      const reader = new FileReader();
      reader.onload = (event: any) => {
        const data: Data = JSON.parse(event.target.result);
        // console.log('Data loaded:', data);
        this.data = data;

        this.queriesByAudioItems = {};
        let queryId = 0;
        let moraId = 0;

        for (let audioItemId in data.talk.audioItems) {
          const audioItem = data.talk.audioItems[audioItemId];
          let text = audioItem.text;
          let moraList: DisplayMora[] = [];

          for (let accentPhrase of audioItem.query.accentPhrases) {
            for (let mora of accentPhrase.moras) {
              moraList.push({
                id: moraId++,
                text: mora.text,
                noteValue: 8,
              });
            }
            if (accentPhrase.pauseMora) {
              moraList.push({
                id: moraId++,
                text: accentPhrase.pauseMora.text,
                noteValue: 8,
              });
            }
          }

          if (!this.queriesByAudioItems[audioItemId]) {
            this.queriesByAudioItems[audioItemId] = [];
          }

          this.queriesByAudioItems[audioItemId].push({
            id: queryId++,
            text: text,
            moras: moraList,
          });
        }
      };
      reader.readAsText(file);
    },
    updateNoteValue(audioItemId: string, event: Event) {
      const newNoteValue = Number((event.target as HTMLSelectElement).value);
      const queries = this.queriesByAudioItems[audioItemId];
      if (queries) {
        queries.forEach(query => {
          query.moras.forEach(mora => {
            mora.noteValue = Number(newNoteValue);
          });
        });
      }
    },
  },
})
</script>

<style scoped>
</style>