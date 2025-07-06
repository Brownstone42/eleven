<template>
    <section class="section">
        <div class="container">
            <h1 class="title">🎙️ AI Voice Conversation</h1>

            <div class="field">
                <label class="label">Enter Text (for ElevenLabs)</label>
                <div class="control">
                    <textarea
                        class="textarea"
                        v-model="text"
                        placeholder="Type something to convert to speech via ElevenLabs..."
                        rows="3"
                    ></textarea>
                </div>
            </div>

            <div class="field">
                <div class="control">
                    <button class="button is-link" :disabled="loading" @click="convertText">
                        {{ loading ? 'Converting...' : 'Convert' }}
                    </button>

                    <button class="button is-link ml-4" @click="startConversation">
                        {{ 'Start (ElevenLab)' }}
                    </button>

                    <button class="button is-link ml-4" @click="stopConversation">
                        {{ 'Stop (ElevenLab)' }}
                    </button>

                    <button class="button is-link ml-4"
                        @click="startVoiceConversation"
                        :disabled="isListening || loading || isSpeaking || (webSocket && webSocket.readyState === 1)">
                        {{ isListening ? 'Listening...' : (loading ? 'AI Thinking...' : (isSpeaking ? 'AI Speaking...' : 'Start AI Conversation')) }}
                    </button>

                    <button class="button is-link ml-4"
                        @click="stopVoiceConversation"
                        :disabled="!webSocket || webSocket.readyState !== 1">
                        Stop AI Conversation
                    </button>
                </div>
            </div>

            <div class="field" v-if="currentInterimTranscript || sttError">
                <label class="label">Your Live Transcription:</label>
                <div class="control">
                    <textarea class="textarea" :value="currentInterimTranscript" readonly rows="2"></textarea>
                </div>
                <p class="help is-danger" v-if="sttError">{{ sttError }}</p>
            </div>

            <div class="field">
                <label class="label">Conversation History:</label>
                <div class="control" style="max-height: 400px; overflow-y: auto; background-color: #f9f9f9; padding: 10px; border-radius: 4px;">
                    <div v-for="(message, index) in chatHistory" :key="index"
                         :class="{'has-text-right': message.type === 'user', 'has-text-left': message.type === 'ai'}"
                         style="margin-bottom: 8px;">
                        <span :class="{'tag is-info is-light': message.type === 'user', 'tag is-primary is-light': message.type === 'ai'}"
                              style="padding: 8px 12px; border-radius: 16px;">
                            {{ message.text }}
                        </span>
                    </div>
                    <!-- Display interim transcription as user speaks (if available and AI is not speaking/thinking) -->
                    <div v-if="currentInterimTranscript && !isSpeaking && isListening" class="has-text-right" style="margin-bottom: 8px;">
                        <span class="tag is-info is-light" style="padding: 8px 12px; border-radius: 16px; opacity: 0.7;">
                            {{ currentInterimTranscript }}...
                        </span>
                    </div>
                    <!-- Display AI thinking status -->
                    <div v-if="loading" class="has-text-left" style="margin-bottom: 8px;">
                        <span class="tag is-primary is-light" style="padding: 8px 12px; border-radius: 16px; opacity: 0.7;">
                            AI Thinking...
                        </span>
                    </div>
                </div>
            </div>

            <div class="field" v-if="audioUrl && !isSpeaking">
                <label class="label">Playback (ElevenLabs)</label>
                <div class="control">
                    <audio :src="audioUrl" controls autoplay></audio>
                </div>
            </div>

            <audio ref="aiAudioPlayer" style="display: none;"></audio>
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
            isListening: false,
            isSpeaking: false,
            transcribedText: '', // To store the text from STT
            sttError: '', // To show STT-related errors

            // --- New properties for WebSocket and streaming ---
            webSocket: null,
            audioContext: null,
            mediaStreamSource: null,
            processor: null,

            chatHistory: [],
            currentInterimTranscript: ''
        }
    },
    beforeUnmount() {
        this.stopVoiceConversation()
    },
    methods: {
        base64ToBlob(base64, mimeType) {
            const byteCharacters = atob(base64)
            const byteNumbers = new Array(byteCharacters.length)
            for (let i = 0; i < byteCharacters.length; i++) {
                byteNumbers[i] = byteCharacters.charCodeAt(i)
            }
            const byteArray = new Uint8Array(byteNumbers)
            return new Blob([byteArray], { type: mimeType })
        },
        async startVoiceConversation() {
            try {
                if (this.webSocket && this.webSocket.readyState === WebSocket.OPEN) {
                    console.log('AI Voice Conversation already started.')
                    return
                }

                this.sttError = ''
                this.transcribedText = ''
                this.currentInterimTranscript = ''
                this.chatHistory = []
                this.loading = false
                this.isSpeaking = false

                // 1. Establish WebSocket Connection
                this.webSocket = new WebSocket('ws://localhost:3001')

                this.webSocket.onopen = () => {
                    console.log('WebSocket connected. Starting audio streaming for AI Voice Conversation.')
                    this.startAudioStreaming() // Start microphone audio capture
                    this.isListening = true // Ready to listen for user input
                }

                this.webSocket.onmessage = async (event) => {
                    const data = JSON.parse(event.data)

                    if (data.transcribedText) {
                        this.currentInterimTranscript = data.transcribedText

                        if (data.isFinal) {
                            this.transcribedText = data.transcribedText
                            console.log('Received FINAL Transcribed Text:', this.transcribedText)

                            // Add user's final utterance to chat history
                            this.chatHistory.push({ type: 'user', text: this.transcribedText })
                            this.currentInterimTranscript = '' // Clear interim after final
                            this.transcribedText = '' // Clear for next turn

                            this.loading = true
                            this.isListening = false
                        }
                    } else if (data.aiAudioBase64) {
                        console.log('Received AI Audio Base64. Playing audio...')
                        this.loading = false // AI finished thinking

                        if (data.aiResponseText) {
                            this.chatHistory.push({ type: 'ai', text: data.aiResponseText })
                        } else {
                            this.chatHistory.push({ type: 'ai', text: 'AI responded with audio.' })
                        }

                        const audioPlayer = this.$refs.aiAudioPlayer
                        //console.log(audioPlayer)

                        if (audioPlayer) {
                            audioPlayer.src = URL.createObjectURL(this.base64ToBlob(data.aiAudioBase64, 'audio/mpeg'))

                            audioPlayer.onplay = () => {
                                this.isSpeaking = true // AI is speaking
                                console.log('AI audio started playing.')
                            }

                            audioPlayer.onended = () => {
                                this.isSpeaking = false
                                this.isListening = true

                                console.log('AI audio finished playing. Ready for next user input.')
                                if (this.audioUrl) {
                                    URL.revokeObjectURL(this.audioUrl)
                                    this.audioUrl = null
                                }
                            }

                            audioPlayer.onerror = (e) => {
                                console.error('Audio playback error:', e)
                                this.sttError = 'Error playing AI audio.'
                                this.isSpeaking = false
                                this.isListening = true
                            }
                            audioPlayer.play()
                        }
                    } else if (data.aiResponseText) {
                        console.log('Received AI Text Response (no audio yet):', data.aiResponseText)
                        this.chatHistory.push({ type: 'ai', text: data.aiResponseText })
                        this.loading = false
                        this.isListening = true
                    } else if (data.error) {
                        console.error('WebSocket Error from Backend:', data.error)
                        this.sttError = `Backend error: ${data.error}`
                        this.loading = false
                        this.isListening = true
                    }
                }

                this.webSocket.onclose = () => {
                    console.log('WebSocket disconnected.')
                    this.stopAudioStreaming()
                    this.isListening = false
                    this.loading = false
                    this.isSpeaking = false // Ensure speaking state is reset
                }

                this.webSocket.onerror = (error) => {
                    console.error('WebSocket error:', error)
                    this.sttError = `WebSocket error: ${error.message || 'Unknown error'}`
                    this.stopAudioStreaming()
                    this.isListening = false
                    this.loading = false
                    this.isSpeaking = false // Ensure speaking state is reset
                }
            } catch (err) {
                console.error('Error starting voice conversation:', err)
                this.sttError = `Error: ${err.message}`
                this.isListening = false
                this.loading = false
                this.isSpeaking = false
            }
        },
        async stopVoiceConversation() {
            if (this.webSocket && this.webSocket.readyState === WebSocket.OPEN) {
                this.webSocket.close()
            }

            this.stopAudioStreaming() // Ensure microphone is off
            this.isListening = false
            this.isSpeaking = false
            this.loading = false
            this.transcribedText = ''
            this.currentInterimTranscript = ''
            this.sttError = ''
            this.chatHistory = []

            const aiAudioPlayer = this.$refs.aiAudioPlayer

            if (aiAudioPlayer) {

                aiAudioPlayer.onplay = null
                aiAudioPlayer.onended = null
                aiAudioPlayer.onerror = null

                if (!aiAudioPlayer.paused) {
                    aiAudioPlayer.pause()
                    aiAudioPlayer.currentTime = 0
                }
                if (aiAudioPlayer.src) {
                    URL.revokeObjectURL(aiAudioPlayer.src)
                    aiAudioPlayer.src = ''
                }
            }

            if (this.audioUrl) {
                URL.revokeObjectURL(this.audioUrl)
                this.audioUrl = null
            }

            console.log('Conversation session fully ended.')
        },
        async startAudioStreaming() {
            try {
                // Try to set the desired sample rate for the AudioContext
                this.audioContext = new (window.AudioContext || window.webkitAudioContext)({
                    sampleRate: 16000,
                })
                const stream = await navigator.mediaDevices.getUserMedia({ audio: true })
                this.mediaStreamSource = this.audioContext.createMediaStreamSource(stream)

                // Create a ScriptProcessorNode to get raw audio data
                // Buffer size must be one of: 256, 512, 1024, 2048, 4096, 8192, 16384
                // We'll use 4096. Input channels: 1 (mono), Output channels: 1 (mono)
                this.processor = this.audioContext.createScriptProcessor(4096, 1, 1)

                this.processor.onaudioprocess = (event) => {
                    // Only send audio if we are in a listening state and AI is not speaking
                    // This prevents sending audio while AI is generating a response or speaking
                    if (this.isListening && !this.isSpeaking && this.webSocket && this.webSocket.readyState === WebSocket.OPEN) {
                        const inputBuffer = event.inputBuffer.getChannelData(0)
                        const pcm16 = new Int16Array(inputBuffer.length)

                        for (let i = 0; i < inputBuffer.length; i++) {
                            pcm16[i] = Math.max(-1, Math.min(1, inputBuffer[i])) * 0x7FFF
                        }

                        this.webSocket.send(pcm16.buffer)
                    }
                }

                // Connect the audio graph
                this.mediaStreamSource.connect(this.processor)
                this.processor.connect(this.audioContext.destination)

                console.log('Audio streaming to WebSocket started for AI Voice Conversation.')
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
            console.log('Audio streaming stopped for AI Voice Conversation.')
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

<style scoped>
.tag {
    white-space: normal; /* Allow text to wrap */
    height: auto; /* Adjust height based on content */
    padding: 8px 12px; /* More padding for better look */
    line-height: 1.5; /* Improve readability */
    word-wrap: break-word; /* Break long words */
    font-size: 0.9rem; /* Slightly smaller font for chat bubbles */
}
.has-text-right .tag {
    background-color: #e6f7ff; /* Light blue for user */
    color: #1890ff;
    border-radius: 16px 16px 4px 16px; /* Rounded corners with one flat edge */
}
.has-text-left .tag {
    background-color: #f6ffed; /* Light green for AI */
    color: #52c41a;
    border-radius: 16px 16px 16px 4px; /* Rounded corners with one flat edge */
}
/* Ensure the control div for chat history has some padding */
.control[style*="max-height: 400px"] {
    padding: 15px;
}
</style>