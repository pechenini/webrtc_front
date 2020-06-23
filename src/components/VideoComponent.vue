<template>
    <div class="hello">
        Stream
        <div v-if="roomUid" class="">Room UID {{roomUid}}</div>
        <div class="controls">
            <a @click.prevent="startStream" class="control-btn">Start stream</a>
            <a @click.prevent="showRoomUidField = !showRoomUidField" class="control-btn">Connect</a>
        </div>
        <div v-if="showRoomUidField" class="">
            <input type="text" v-model="roomUid">
            <a @click.prevent="connectStream" class="control-btn">Enter room</a>
        </div>
        <div class="">
            <video id="video1" width="480" height="320" autoplay muted controls></video>
            <br/>
        </div>
    </div>
</template>

<script>
    import axios from "axios";

    export default {
        data() {
            return {
                showRoomUidField: false,
                session: "",
                remoteSession: "",
                roomUid: "",
                url: "http://true-tech.php.nixdev.co:9090",
                pc: new RTCPeerConnection({
                    iceServers: [
                        {
                            urls: 'stun:stun.l.google.com:19302'
                        }
                    ]
                })
            }
        },
        created() {
            this.pc.onicecandidate = event => {
                if (event.candidate === null) {
                    this.session = btoa(JSON.stringify(this.pc.localDescription))
                }
            }

            this.pc.oniceconnectionstatechange = () => console.log(this.pc.iceConnectionState)
        },
        methods: {
            async startStream() {
                try {
                    this.showRoomUidField = false
                    const stream = await navigator.mediaDevices.getUserMedia({video: true, audio: true})
                    stream.getTracks().forEach(track => this.pc.addTrack(track, stream))
                    document.getElementById('video1').srcObject = stream;
                    const d = await this.pc.createOffer()
                    await this.pc.setLocalDescription(d)
                    const localDescr = btoa(JSON.stringify(this.pc.localDescription))
                    this.session = localDescr
                    const response = await axios.post(this.url + "/sdp", {
                        description: localDescr
                    })
                    console.log(response)
                    this.remoteSession = response.data.answer;
                    this.roomUid = response.data.room_uid
                    await this.pc.setRemoteDescription(new RTCSessionDescription(JSON.parse(window.atob(response.data.answer))))
                } catch (e) {
                    console.log(e)
                }

            },
            async connectStream() {
                try {
                    this.showRoomUidField = true
                    this.pc.addTransceiver('video')
                    const d = await this.pc.createOffer()

                    await this.pc.setLocalDescription(d)
                    this.session = btoa(JSON.stringify(d))

                    this.pc.addEventListener('track', async (event) => {
                        const el = document.getElementById('video1')
                        el.srcObject = event.streams[0]
                        el.autoplay = true
                        el.controls = true
                        console.log("ontrack")
                    });

                    const response = await axios.post(this.url + "/connect", {
                        description: this.session,
                        room_uid: this.roomUid
                    })
                    console.log(response.data)
                    this.remoteSession = response.data.response
                    await this.pc.setRemoteDescription(new RTCSessionDescription(JSON.parse(window.atob(this.remoteSession))))
                } catch (e) {
                    console.log(e)
                }
            },
        }
    }
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
    .controls {
        display: flex;
        flex-direction: row;
        justify-content: center;
    }

    .control-btn {
        margin: 10px;
        padding: 10px;
        color: #fff;
        font-weight: 900;
        font-size: 20px;
        background: #2db42d;
        cursor: pointer;
    }
</style>
