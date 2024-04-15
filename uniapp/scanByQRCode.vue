<template>
    <view class="page">
        <video
            autoplay
            muted
            ref="video"
            id="uni-video"
            class="uni-video"
            object-fit="cover"
            width="100%"
            height="100%"
            :controls="false"
            :show-center-play-btn="false"
            :show-play-btn="false"
            :direction="0"
        />
        <view class="btn-close" @click="close">
            <van-icon name="cross" />
        </view>
        <view class="line-wrap">
            <view class="line-animation" :style="{ animationPlayState: isStop ? 'paused' : 'running' }">
                <view class="line-shadow"></view>
                <view class="line"> </view>
            </view>
        </view>
    </view>
</template>

<script>
import { BrowserMultiFormatReader } from "@zxing/library";
import { Icon as VanIcon } from "vant";
import axios from "axios";

let codeReader, selectedDeviceId;

export default {
    components: { VanIcon },
    data() {
        return {
            isStop: false,
        };
    },
    beforeDestroy() {
        if (codeReader) codeReader.reset();
    },
    mounted() {
        // uniapp 中的video是额外套了一层uni-video标签，所以需要获取到uni-video标签中的video元素。如果不用 uniapp 可以省略这一步，直接把 id 写到 video 标签上
        const video = document.getElementById("uni-video").getElementsByTagName("video")[0];
        video.setAttribute("id", "video-id");
        codeReader = new BrowserMultiFormatReader();
        codeReader.listVideoInputDevices().then((videoInputDevices) => {
            selectedDeviceId = videoInputDevices[videoInputDevices.length - 1].deviceId;
            this.scan();
        });
    },
    methods: {
        close() {
            uni.navigateBack(1);
        },
        // 扫一扫识别二维码
        scan() {
            this.isStop = false;
            // decodeFromInputVideoDevice只触发一次扫码， 暂停后重新调用decodeFromInputVideoDevice会先黑屏再重新初始化
            // decodeFromInputVideoDeviceContinuously会一直触发扫码，但是暂停后重新调用decodeFromInputVideoDeviceContinuously不会黑屏再重新初始化
            codeReader.decodeFromInputVideoDeviceContinuously(selectedDeviceId, "video-id", (result, err) => {
                // decodeFromInputVideoDeviceContinuously会一直触发扫码，扫码成功后暂停，避免多次触发
                if (result && result.text) {
                    this.isStop = true;
                    this.$refs.video.pause();
                    codeReader.stopContinuousDecode();
                    this.upload(result.text);
                }
            });
        },
        upload(data) {
            uni.showLoading({
                title: "识别中...",
            });
            axios
                .get(`/submit`, { params: { ewmData: btoa(data) } })
                .then(({ data: res }) => {
                    if (res.success) {
                        uni.showToast({
                            title: "识别成功",
                            icon: "none",
                            duration: 2000,
                        });
                        // TODO 成功回调
                    } else {
                        uni.showToast({
                            title: res.message || "识别失败",
                            icon: "none",
                            duration: 2000,
                        });
                        // 重新触发扫码
                        this.scan();
                    }
                })
                .catch((e) => {
                    console.log("e", e);
                    uni.showToast({
                        title: "识别失败",
                        icon: "none",
                        duration: 2000,
                    });
                    // 重新触发扫码
                    this.scan();
                })
                .finally(() => {
                    uni.hideLoading();
                });
        },
    },
};
</script>

<style scoped>
.page {
    display: flex;
    flex-direction: column;
    width: 100vw;
    height: 100vh;
    position: relative;
}
.uni-video {
    width: 100%;
    height: 100%;
    position: fixed;
    left: 0;
    top: 0;
}
.btn-close {
    position: fixed;
    width: 22px;
    height: 22px;
    border-radius: 50em;
    background-color: #fff;
    left: 24px;
    top: 56px;
    color: #333;
    display: flex;
    align-items: center;
    justify-content: center;
}
.line-wrap {
    position: fixed;
    left: 0;
    top: 50%;
    transform: translate(0, -50%);
    padding: 0 32px;
    /* border: 1px solid #f00; */
    width: 100%;
    height: 312px;
}
.line-wrap .line-shadow {
    width: 100%;
    height: 24px;
    background: linear-gradient(180deg, rgba(24, 144, 255, 0) 0%, #1890ff 100%);
    filter: blur(10px);
}
.line-wrap .line {
    height: 1px;
    background: #1890ff;
}

.line-wrap .line-animation {
    animation: scanLine 2s linear infinite;
}

@keyframes scanLine {
    0% {
        -webkit-transform: translate(0, 0);
    }
    100% {
        -webkit-transform: translate(0, 300px);
    }
}
</style>
