<template>
    <div :class="[{'video-box-remote': type === 'remote', 'video-box-local': type === 'local'}, 'video-box-wrap']">
    
        <div :class="[{'controls-ban': type === 'remote'}, 'controls']">
    
            <!-- 操作区域 -->
            <div :class="[ type === 'remote' ? 'controls-button-ban' : 'controls-button', 'controls-button-audio']" @click="handleControlClick('audio')" v-show="user.audio !== undefined || user.audio !== null">
                <span class="media-status audio-open" v-show="user.audio && user.audio.status == 2"></span>
                <span class="media-status audio-off" v-show="!user.audio || user.audio.status != 2"></span>
            </div>
            <div :class="[type === 'remote' ? 'controls-button-ban' : 'controls-button', 'controls-button-video']" @click="handleControlClick('video')" v-show="user.video !== undefined || user.video !== null">
                <span class="media-status camera-open" v-show="user.video && user.video.status == 2"></span>
                <span class="media-status camera-off" v-show="!user.video || user.video.status != 2"></span>
            </div>
        </div>
        <div class="user-info" @click="handleControlBtnClick">
            {{ username }}
            用户名: {{ user.name }} ({{ type === 'remote' ? '远端' : '本地' }})
        </div>
        <div class="source-video-wrap">
            <video autoplay :controls="control" class="source-video" :srcObject="videoStream" v-show="user.video.status == 2"></video>
            <audio :srcObject="audioStream" v-show="false" autoplay></audio>
        </div>
    </div>
</template>

<script>
export default {
    props: {
        type: {
            type: String,
            required: true
        },
        user: {
            type: Object,
            required: true
        }
    },
    data() {
        return {
            control: false,
            videoStream: null,
            audioStream: null
        }
    },
    watch: {
        user: newVal => {
            console.log('+++n: ', newVal)
        }
    },
    methods: {
        /**
         * 根据类型控制操作
         * @param {string} type - 类型
         */
        handleControlClick(sourceType, switchType) {
            const self = this
            if(self.user && self.user.id === room.selfUser.id){
                if(sourceType === 'video'){
                    if(self.user.video.status === StreamStatus.published){
                        self.$emit('stream-switch', {sourceType: 'video', switchType: 'close'})
                        return
                    }

                    if(self.user.video.status === StreamStatus.init){
                        self.$emit('stream-switch', {sourceType: 'video', switchType: 'open'})
                        return
                    }
                }
                if(sourceType === 'audio'){
                    if(self.user.audio.status === StreamStatus.published){
                        self.$emit('stream-switch', {sourceType: 'audio', switchType: 'mute'})
                        return
                    }

                    if(self.user.audio.status === StreamStatus.init){
                        self.$emit('stream-switch', {sourceType: 'audio', switchType: 'open'})
                        return
                    }

                    if(self.user.audio.status === StreamStatus.muted){
                        self.$emit('stream-switch', {sourceType: 'audio', switchType: 'unmute'})
                        return
                    }
                }
            }
        },
        renderStream(type, stream) {
            switch(type){
                case 'audio':
                    this.audioStream = stream
                    break
                case 'video':
                    this.videoStream = stream
            }
        }
    }
}
</script>

<style scoped>
.video-box-wrap {
    background-color: rgba(36, 59, 85, 0.75);
    background-image: url('../../../lib/images/buddy.svg');
    background-size: auto 80%;
    background-repeat: no-repeat;
    background-position: bottom center;
    border: 2px solid rgba(36, 59, 85, 0.75);
    position: absolute;
    transition: all ease .3s;
    box-shadow: 0 5px 16px 2px rgb(17 17 17 / 5);
}

.video-box-wrap:hover {
    border: 2px solid #068511;
}

.video-box-wrap-voice-active {
    border: 2px solid #068511;
}

.video-box-remote {
    width: 100%;
    height: 100%;
    left: 0;
    top: 0;
}

.video-box-local {
    width: 292px;
    height: 248px;
    right: 20px;
    bottom: 20px;
}

.controls-ban, .controls{
    position: absolute;
    z-index: 1000;
    top: 0;
    left: 0;
    right: 0;
    display: flex;
    flex-direction: row;
    align-items: center;
    padding: 10px;
}

.controls-button {
    width: 32px;
    height: 32px;
    display: inline-flex;
    opacity: .85;
    background: rgba(0, 0, 0, .5);
    margin-right: 10px;
    justify-content: center;
    align-items: center;
    position: relative;
    z-index: 1001;
    transition: all ease .3s;
}

.controls-button-ban{
    cursor:not-allowed;
    width: 32px;
    height: 32px;
    display: inline-flex;
    opacity: .85;
    background: rgba(0, 0, 0, .5);
    margin-right: 10px;
    justify-content: center;
    align-items: center;
    position: relative;
    z-index: 1001;
}
.controls-button:hover{
    cursor: pointer;
    background: rgba(0, 0, 0, 0.8);
}

.user-info {
    position: absolute;
    left: 0;
    bottom: 0;
    width: 100%;
    height: 30px;
    background: #00000033;
    color: #ffffff;
    line-height: 30px;
    box-sizing: border-box;
    font-size: .8rem;
    padding: 0 10px;
    backdrop-filter: blur(5px);
    z-index: 1000;
}

.source-video-wrap {
    position: relative;
    width: 100%;
    height: 100%;
    z-index: 998;
}

.source-video {
    position: relative;
    width: 100%;
    height: 100%;
    z-index: 999;
}

.media-status {
    display: block;
    width: 26px;
    height: 26px;
}

.audio-open {
    background: url('../assets/images/audio_open.svg') no-repeat center center;
    background-size: contain;    
}
.audio-off {
    background: url('../assets/images/audio_off.svg') no-repeat center center;
    background-size: contain;
}   
.camera-open {
    background: url('../assets/images/camera_open.svg') no-repeat center center;
    background-size: contain;
}
.camera-off {
    background: url('../assets/images/camera_off.svg') no-repeat center center;
    background-size: contain;
}
</style>