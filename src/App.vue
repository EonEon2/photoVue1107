<template>
  <div>
    <h1>Camera Feed on Canvas</h1>
    <video ref="videoElement" width="640" height="480" autoplay></video>
    <canvas ref="canvasElement" width="640" height="480"></canvas>
    <button @click="toggle">{{ isToggled ? 'Stop Camera' : 'Start Camera' }}</button>
    <button @click="switchCamera">Switch Camera</button>
    <button @click="photosend" v-if="isToggled">Photo And Send</button>
    <div v-if="base64Image">
      <h3>Base64 Encoded Image:</h3>
      <textarea v-model="base64Image" rows="6" cols="80" readonly></textarea>
    </div>
  </div>
</template>

<script>
import { ref, onBeforeUnmount } from "vue";
import axios from "axios";

export default {
  setup() {
    // reactive 변수
    const videoStream = ref(null); // 카메라 스트림 저장
    const isToggled = ref(false); // 카메라 상태 (켜짐/꺼짐)
    const currentDevice = ref('environment'); // 초기 카메라 디바이스: 후면 카메라
    const base64Image = ref(''); // Base64 이미지 데이터를 저장

    // 카메라 스트림을 시작하는 함수
    const startCamera = async () => {
      const stream = await navigator.mediaDevices.getUserMedia({
        video: { facingMode: currentDevice.value },
      });
      videoStream.value = stream;
      // 비디오 스트림을 video 요소에 연결
      const videoElement = document.querySelector("video");
      videoElement.srcObject = stream;
      // 비디오가 시작되면 캔버스에 그리기 시작
      videoElement.onplaying = drawToCanvas;
    };

    // 카메라 스트림을 멈추는 함수
    const stopCamera = () => {
      if (videoStream.value) {
        const tracks = videoStream.value.getTracks();
        tracks.forEach((track) => track.stop()); // 트랙을 정지시켜 카메라 끄기
        const videoElement = document.querySelector("video");
        videoElement.srcObject = null; // video 요소에서 스트림을 제거
      }
    };

    // 비디오를 캔버스에 그리는 함수
    const drawToCanvas = () => {
      const videoElement = document.querySelector("video");
      const canvasElement = document.querySelector("canvas");
      const ctx = canvasElement.getContext("2d");

      // 애니메이션을 통해 비디오를 캔버스에 그리기
      const render = () => {
        ctx.drawImage(videoElement, 0, 0, canvasElement.width, canvasElement.height);
        requestAnimationFrame(render); // 계속해서 캔버스에 그리기
      };

      render();
    };

    // 카메라를 전면/후면으로 전환하는 함수
    const switchCamera = () => {
      currentDevice.value = currentDevice.value === 'environment' ? 'user' : 'environment';
      if (isToggled.value) {
        stopCamera(); // 현재 카메라 끄기
        startCamera(); // 새로운 카메라로 스트림 시작
      }
    };

    const takePhoto = () => {
      const canvasElement = document.querySelector("canvas");
      // 캔버스에서 현재 프레임을 Base64 이미지 URL로 추출
      const imageUrl = canvasElement.toDataURL('image/webp'); // 'image/webp' 대신 'image/jpeg'로 변경
      // Base64 인코딩된 이미지를 문자열로 저장
      base64Image.value = imageUrl.split(",")[1]; // data:image/jpeg;base64, 부분을 제거하고 Base64 값만 저장
      console.log("Updated base64Image:", base64Image.value);
    };

    // 서버로 이미지를 전송하는 함수
    const sendServer = async () => {
      if (!base64Image.value) {
        console.error("No image to send.");
        return;
      }

      console.log("Sending base64 image to server:", base64Image.value);
      try {
        const response = await axios.post('http://localhost:8080/api/v1/saveState', {
          image: base64Image.value, // 서버에 전송할 데이터 형식을 명확히 정의
        });
        console.log("Response from server:", response.data);
      } catch (err) {
        console.error("Server request failed:", err);
      }
    };

    // 토글 버튼 클릭 시 카메라 상태 전환
    const toggle = () => {
      if (isToggled.value) {
        // 카메라 끄기
        stopCamera();
      } else {
        // 카메라 켜기
        startCamera();
      }
      isToggled.value = !isToggled.value; // 상태 변경
    };

    // 사진을 찍고 서버로 전송하는 함수
    const photosend = async () => {
      await takePhoto(); // 사진을 찍고 base64 값이 최신화 됨
      console.log("Final base64 value after photo:", base64Image.value);
      await sendServer(); // 서버로 전송
    };

    // 컴포넌트가 파괴되기 전에 카메라 스트림을 멈추는 함수
    onBeforeUnmount(() => {
      if (videoStream.value) {
        const tracks = videoStream.value.getTracks();
        tracks.forEach((track) => track.stop());
      }
    });

    return {
      isToggled,
      base64Image,
      toggle,
      switchCamera,
      photosend,
    };
  },
};
</script>

<style scoped>
h1 {
  text-align: center;
}

video,
canvas {
  display: block;
  margin: 0 auto;
}

button {
  display: block;
  margin: 10px auto;
  padding: 10px 20px;
  font-size: 16px;
  cursor: pointer;
}

textarea {
  width: 100%;
  margin: 10px 0;
}

a {
  display: none; /* 다운로드 링크는 화면에 표시되지 않음 */
}
</style>
