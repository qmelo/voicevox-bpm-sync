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
      
      <div v-for="query in queries" :key="query.id" class="my-4 w-full">
        <div>{{ query.text }}</div>
          <div class="flex flex-wrap">
            <div v-for="mora in query.moras" :key="mora.id" class="flex flex-col items-center my-2">
              <span>{{ mora.text }}</span>
              <select v-model="mora.noteType" class="select select-bordered ml-2">
                <option v-for="option in noteOptions" :key="option.value" :value="option.value">{{ option.label }}</option>
              </select>
            </div>
          </div>
      </div>

      <button @click="convert" class="btn btn-primary mt-4 mb-4 w-full max-w-xs">プロジェクトファイルを変換</button>
      
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue'

interface Mora {
  id: number;
  text: string;
  consonantLength?: number;
  vowelLength?: number;
  noteType: number;
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
      queries: [] as any[],
      noteOptions: [
        { value: 2, label: '1/4' },
        { value: 3, label: '1/4T' },
        { value: 4, label: '1/8' },
        { value: 6, label: '1/8T' },
        { value: 8, label: '1/16' },
        { value: 12, label: '1/16T' },
        { value: 16, label: '1/32' },
      ],
      fileContent: null as string | null,
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
    convert(event: any) {

      if (!this.fileContent) return;
      
      const data: Data = JSON.parse(this.fileContent);
      const bpm = this.bpm;

      console.log('Data loaded:', data);

      for (let audioItem of Object.values(data.talk.audioItems)) {
        console.log('Processing audioItem:', audioItem);

        for (let accentPhrase of audioItem.query.accentPhrases) {
          console.log('Processing accentPhrase:', accentPhrase);

          const updateMora = function (mora: Mora) {
            const noteLength = 60 / bpm / mora.noteType;
            console.log('Get noteType:', mora.noteType);
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
            console.log(`Original mora:`, mora);
            updateMora(mora);
            console.log(`Updated mora:`, mora);
          }

          if (accentPhrase.pauseMora) {
            updateMora(accentPhrase.pauseMora);
          }

        }
      }

      const blob = new Blob([JSON.stringify(data)], { type: "text/plain" });
      const link = document.createElement("a");
      link.href = URL.createObjectURL(blob);
      link.download = 'converted_project.vvproj';
      link.click();
    },
    display(event: any){
      const file = event.target.files[0];
      const reader = new FileReader();
      reader.onload = (event: any) => {
        const data: Data = JSON.parse(event.target.result);
        console.log('Data loaded:', data);
        this.fileContent = event.target.result;

        this.queries = [];
        let queryId = 0;
        let moraId = 0;

        for (let audioItem of Object.values(data.talk.audioItems)) {
          let text = audioItem.text;
          let moras = [];

          for (let accentPhrase of audioItem.query.accentPhrases) {
            for (let mora of accentPhrase.moras) {
              moras.push({
                id: moraId++,
                text: mora.text,
                noteType: 8,
                consonantLength: mora.consonantLength,
                vowelLength: mora.vowelLength,
              });
            }
          }

          this.queries.push({
            id: queryId++,
            text: text,
            moras: moras,
          });
        }
      };
      reader.readAsText(file);
    },
  },
})
</script>

<style scoped>
</style>