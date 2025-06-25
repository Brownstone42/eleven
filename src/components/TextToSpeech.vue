<template>
    <section class="section">
        <div class="container">
            <h1 class="title">🎙️ Text to Speech</h1>

            <div class="field">
                <label class="label">Enter Text</label>
                <div class="control">
                    <textarea class="textarea" v-model="text" 
                        placeholder="Type something to convert to speech..." rows="5"
                    ></textarea>
                </div>
            </div>

            <div class="field">
                <div class="control">
                    <button class="button is-link" :disabled="loading" @click="convertText">
                        {{ loading ? 'Converting...' : 'Convert' }}
                    </button>

                    <button class="button is-link ml-4" :disabled="loading2" @click="conversation">
                        {{ loading2 ? 'Preparing...' : 'Talk' }}
                    </button>
                </div>
            </div>

            <div class="field" v-if="audioUrl">
                <label class="label">Playback</label>
                <div class="control">
                    <audio :src="audioUrl" controls autoplay></audio>
                </div>
            </div>

        </div>
    </section>
</template>

<script>
export default {
    name: 'TextToSpeech',
    data() {
        return {
            text: '',
            audioUrl: null,
            loading: false,
            loading2: false,
        }
    },
    methods: {
        async conversation() {
            if (!this.text.trim()) {
                alert('Please enter some text.')
                return
            }

            this.loading2 = true

            try {
                
            } catch (error) {
                alert(error.message || 'Something went wrong.')
            } finally {
                this.loading2 = false
            }
        },
        async convertText() {
            if (!this.text.trim()) {
                alert('Please enter some text.')
                return
            }

            this.loading = true

            try {
                const apiKey = import.meta.env.VITE_ELEVENLABS_API_KEY
                const voiceId = 'EXAVITQu4vr4xnSDxMaL'

                const response = await fetch(`https://api.elevenlabs.io/v1/text-to-speech/${voiceId}`, {
                    method: 'POST',
                    headers: {
                        'xi-api-key': apiKey,
                        'Content-Type': 'application/json',
                        'Accept': 'audio/mpeg',
                    },
                    body: JSON.stringify({
                        text: this.text,
                        model_id: 'eleven_multilingual_v2',
                        outputFormat: "mp3_44100_128",
                        voice_settings: {
                            stability: 0.5,
                            similarity_boost: 0.75,
                        }
                    })
                })

                const blob = await response.blob()
                this.audioUrl = URL.createObjectURL(blob)
            } catch (error) {
                alert(error.message || 'Something went wrong.')
            } finally {
                this.loading = false
            }
        }
    }
}
</script>