<template>
    <section class="section">
        <div class="container">
            <h1 class="title">🎙️ Text to Speech</h1>

            <div class="field">
                <label class="label">Enter Text</label>
                <div class="control">
                    <textarea
                        class="textarea"
                        v-model="text"
                        placeholder="Type something to convert to speech..."
                        rows="5"
                    ></textarea>
                </div>
            </div>

            <div class="field">
                <div class="control">
                    <button class="button is-link" :disabled="loading" @click="convertText">
                        {{ loading ? 'Converting...' : 'Convert' }}
                    </button>

                    <button class="button is-link ml-4" @click="startConversation">
                        {{ 'Start Conversation' }}
                    </button>

                    <button class="button is-link ml-4" @click="stopConversation">
                        {{ 'Stop Conversation' }}
                    </button>

                    <button class="button is-link ml-4" @click="talk">
                        {{ isRecording ? 'Stop Talking' : 'Talk' }}
                    </button>
                </div>
            </div>

            <div class="field" v-if="transcribedText || sttError">
                <label class="label">Transcribed Text</label>
                <div class="control">
                    <textarea
                        class="textarea"
                        :value="transcribedText"
                        readonly
                        rows="3"
                    ></textarea>
                </div>
                <p class="help is-danger" v-if="sttError">{{ sttError }}</p>
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
import { Conversation } from '@elevenlabs/client'

export default {
    name: 'TextToSpeech',
    data() {
        return {
            text: '',
            audioUrl: null,
            loading: false,
            conversation: null,

            // --- New properties for STT ---
            mediaRecorder: null,
            audioChunks: [],
            isRecording: false, // To track recording state
            transcribedText: '', // To store the text from STT
            sttError: '', // To show STT-related errors

            // --- New properties for WebSocket and streaming ---
            webSocket: null,
            audioContext: null,
            mediaStreamSource: null,
            processor: null,
        }
    },
    beforeUnmount() {
        this.stopAudioStreaming()
        if (this.webSocket && this.webSocket.readyState === WebSocket.OPEN) {
            this.webSocket.close()
        }
    },
    methods: {
        async talk() {
            try {
                if (!this.isRecording) {
                    // --- Start Recording ---
                    this.sttError = '' // Clear previous errors
                    this.transcribedText = '' // Clear previous text
                    this.audioChunks = [] // Clear previous audio chunks

                    this.webSocket = new WebSocket('ws://localhost:3001')

                    this.webSocket.onopen = () => {
                        console.log('WebSocket connected.')
                        this.isRecording = true // Set recording state after connection is open
                        // Start audio processing only after WebSocket is open
                        this.startAudioStreaming()
                    }

                    this.webSocket.onmessage = (event) => {
                        // This is where we'll receive transcribed text from the backend
                        const data = JSON.parse(event.data)
                        if (data.transcribedText) {
                            this.transcribedText = data.transcribedText
                            //console.log('Received Transcribed Text:', this.transcribedText)
                        }
                    }

                    this.webSocket.onclose = () => {
                        console.log('WebSocket disconnected.')
                        this.stopAudioStreaming() // Ensure audio streaming stops
                        this.isRecording = false // Reset recording state
                    }

                    this.webSocket.onerror = (error) => {
                        console.error('WebSocket error:', error)
                        this.sttError = `WebSocket error: ${error.message || 'Unknown error'}`
                        this.stopAudioStreaming() // Ensure audio streaming stops
                        this.isRecording = false // Reset recording state
                    }
                } else {
                    // --- Stop Recording & Close WebSocket ---
                    this.stopAudioStreaming()

                    if (this.webSocket && this.webSocket.readyState === WebSocket.OPEN) {
                        this.webSocket.close()
                    }

                    this.isRecording = false
                    console.log('Stopped talking and closed WebSocket.')
                }
            } catch (err) {
                console.error('Error starting talk session:', err)
                this.sttError = `Error: ${err.message}`
                this.isRecording = false
            }
        },
        async startAudioStreaming() {
            try {
                // Try to set the desired sample rate for the AudioContext
                this.audioContext = new (window.AudioContext || window.webkitAudioContext)({
                    sampleRate: 16000,
                }) // <--- ADD THIS
                const stream = await navigator.mediaDevices.getUserMedia({ audio: true })
                this.mediaStreamSource = this.audioContext.createMediaStreamSource(stream)

                // Create a ScriptProcessorNode to get raw audio data
                // Buffer size must be one of: 256, 512, 1024, 2048, 4096, 8192, 16384
                // We'll use 4096. Input channels: 1 (mono), Output channels: 1 (mono)
                this.processor = this.audioContext.createScriptProcessor(4096, 1, 1)

                this.processor.onaudioprocess = (event) => {
                    if (
                        !this.isRecording ||
                        !this.webSocket ||
                        this.webSocket.readyState !== WebSocket.OPEN
                    ) {
                        return
                    }

                    // Get the raw audio data (Float32Array) from the input buffer
                    const inputBuffer = event.inputBuffer.getChannelData(0)
                    /*console.log(
                        'Audio inputBuffer length:',
                        inputBuffer.length,
                        'First value:',
                        inputBuffer[0],
                    )*/

                    // Convert Float32Array to 16-bit PCM (Int16Array)
                    // Google STT expects LINEAR16, which is 16-bit signed integers
                    const pcm16 = new Int16Array(inputBuffer.length)
                    for (let i = 0; i < inputBuffer.length; i++) {
                        pcm16[i] = Math.max(-1, Math.min(1, inputBuffer[i])) * 0x7fff // Scale to 16-bit range
                    }

                    //console.log('PCM16 buffer length:', pcm16.length, 'First value:', pcm16[0])

                    if (this.webSocket.readyState === WebSocket.OPEN) {
                        this.webSocket.send(pcm16.buffer)
                        //console.log('Sent audio chunk to WebSocket.')
                    }
                }

                // Connect the audio graph
                this.mediaStreamSource.connect(this.processor)
                this.processor.connect(this.audioContext.destination)

                console.log('Audio streaming started.')
            } catch (err) {
                console.error('Error starting audio streaming:', err)
                this.sttError = `Audio streaming error: ${err.message}`
                this.stopAudioStreaming()
                if (this.webSocket && this.webSocket.readyState === WebSocket.OPEN) {
                    this.webSocket.close()
                }
            }
        },
        stopAudioStreaming() {
            if (this.processor) {
                this.processor.disconnect()
                this.processor.onaudioprocess = null // Remove event listener
                this.processor = null
            }
            if (this.mediaStreamSource) {
                this.mediaStreamSource.disconnect()
                // Stop the microphone track
                if (this.mediaStreamSource.mediaStream) {
                    this.mediaStreamSource.mediaStream.getTracks().forEach((track) => track.stop())
                }
                this.mediaStreamSource = null
            }
            if (this.audioContext) {
                this.audioContext.close()
                this.audioContext = null
            }
            console.log('Audio streaming stopped.')
        },
        async startConversation() {
            try {
                const res = await fetch('http://localhost:3001/api/get-signed-url')
                console.log(res)

                const { signedUrl } = await res.json()
                console.log(signedUrl)

                this.conversation = await Conversation.startSession({
                    signedUrl,
                    onMessage: (msg) => {
                        console.log('🗨️ Message:', msg)
                    },
                    onAudio: (audioChunk) => {
                        //console.log(audioChunk)
                    },
                    onError: (err) => {
                        console.error('❌ Conversation error:', err)
                    },
                })

                console.log('✅ Connected to conversation.')
            } catch (err) {
                console.error('Failed to start conversation:', err)
            }
        },
        async stopConversation() {
            if (this.conversation) {
                await this.conversation.endSession()
                console.log('Conversation closed.')
                this.conversation = null
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
                const voiceId = 'oglvHTli1vyPUK5VxwZY'

                const response = await fetch(
                    `https://api.elevenlabs.io/v1/text-to-speech/${voiceId}`,
                    {
                        method: 'POST',
                        headers: {
                            'xi-api-key': apiKey,
                            'Content-Type': 'application/json',
                            Accept: 'audio/mpeg',
                        },
                        body: JSON.stringify({
                            text: this.text,
                            model_id: 'eleven_multilingual_v2',
                            outputFormat: 'mp3_44100_128',
                            voice_settings: {
                                stability: 0.5,
                                similarity_boost: 0.75,
                            },
                        }),
                    },
                )

                const blob = await response.blob()
                this.audioUrl = URL.createObjectURL(blob)
            } catch (error) {
                alert(error.message || 'Something went wrong.')
            } finally {
                this.loading = false
            }
        },
    },
}
</script>
