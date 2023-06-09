<template>
  <div class="analyze-container">
    <div class="analyze-analyze">
      <div class="box">
        <div>
          <img src="../assets/maltese.jpg" alt="real-1" class="real-1" />
          <img src="../assets/sample_icon1.png" alt="icon1" class="icon-1" />
          <div class="text-group">
            <div class="main-text"><span>DogMoji</span></div>
            <div class="main-text2"><span>Make your dog an emoji!</span></div>
            <b-button
              variant="warning"
              class="button"
              size="md"
              @click="scrollToUpload"
              >Let's start!
            </b-button>
          </div>
          <img src="../assets/sample2.jpg" alt="real-2" class="real-2" />
          <div class="icon-2"></div>
          <img src="../assets/sample3.jpg" alt="real-3" class="real-3" />
          <div class="icon-3"></div>
        </div>
        <div class="container" ref="fileUpload" tabindex="0">
          <div v-if="!imageFile">
            <h3>Upload Image</h3>
            <div class="drop_box">
              <header>
                <h4>Select file here</h4>
              </header>
              <p>Files Supported: jpg, jpeg, png</p>
              <b-form-file
                v-model="imageFile"
                placeholder="Choose a file or drop it here..."
                drop-placeholder="Drop file here..."
                :state="Boolean(imageFile)"
                accept=".jpg, .png, .jpeg"
                class="rounded-pill bg-light text-dark border-0 py-3 px-4 my-3 form"
                @change="previewImage"
              >
              </b-form-file>
            </div>
          </div>
          <div v-else class="input-image">
            <div style="text-align: center">
              <b-img :src="previewImageData" thumbnail></b-img>
            </div>
            <div style="text-align: center; padding-top: 5px">
              <b-button
                v-if="!loading"
                variant="warning"
                type="button"
                @click="analyzeImage"
                class="rounded-pill px-4 m-1"
                >Analyze
              </b-button>
              <b-button
                v-else
                variant="warning"
                type="button"
                @click="analyzeImage"
                class="rounded-pill px-4 m-1"
                disabled
                >Analyze
                <b-spinner small></b-spinner>
              </b-button>
              <b-button
                variant="secondary"
                type="button"
                @click="resetInput"
                class="rounded-pill px-4 m-1"
                >Reset</b-button
              >
            </div>
          </div>
        </div>
        <div class="result" ref="analyzeResult" tabindex="0">
          <div v-show="result">
            <h2>Result</h2>
            <b-card>
              <b-card-img :src="imageUrl" alt="image"></b-card-img>
            </b-card>
            <b-card class="mt-3" header="Breed">
              <b-card-text class="m-0">{{ breedName }}</b-card-text>
            </b-card>
            <b-card class="mt-3" header="Property">
              <b-card-text class="m-0">{{ breedProp }}</b-card-text>
            </b-card>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import axios from "axios";
import imageCompression from "browser-image-compression";

export default {
  name: "Analyze",
  data() {
    return {
      imageFile: null,
      result: false,
      previewImageData: null,
      responseJson: null,
      responseImage: null,
      imageUrl: null,
      breedName: null,
      breedProp: null,
      image: null,
      loading: false,
    };
  },
  props: {},
  mounted() {
    this.$nextTick(() => {
      this.scrollToResult();
    });
  },
  watch: {
    result() {
      this.scrollToResult();
    },
  },
  methods: {
    previewImage(event) {
      // 업로드한 이미지 화면에 띄울 수 있도록 함
      var input = event.target;
      if (input.files && input.files[0]) {
        var reader = new FileReader();
        reader.onload = (e) => {
          this.previewImageData = e.target.result;
        };
        reader.readAsDataURL(input.files[0]);
      }
    },
    resetInput() {
      // 업로드한 이미지, 분석 결과 등 전부 초기화
      this.previewImageData = null;
      this.imageFile = null;
      this.result = null;
      this.result = false;
      this.loading = false;
    },
    async compressImage(file) {
      const compressedFile = await imageCompression(file, {
        maxSizeKB: 50, // 최대 파일 크기 (메가바이트)
        maxWidthOrHeight: 800, // 이미지의 최대 너비 또는 높이 (픽셀)
      });
      return compressedFile;
    },
    async analyzeImage() {
      // analyze 버튼 클릭 시 수행. 업로드한 이미지를 각 서버에 전송해 결과값을 받아오는 함수
      this.breedName = null;
      this.breedProp = null;
      this.loading = true;

      const formData = new FormData();
      const compressedImage = await this.compressImage(this.imageFile);
      formData.append("file", compressedImage);
      try {
        this.responseImage = await axios.post(
          // emoji 구현
          "https://dogiconde.duckdns.org/final",
          formData
        );
      } catch {
        console.log("error");
      }
      this.imageUrl = this.responseImage.data.image_url;

      const responseBreed = await axios.post(
        // 종 이름 예측
        "https://dogiconde.duckdns.org/predict",
        formData
      );
      this.breedName = responseBreed.data.breed;
      console.log(this.breedName);
      await this.makeResult(); // this.breedName에 종 이름 선언했으니, makeResult() 호출 가능

      // 앞선 모든 과정이 오류없이 끝난다면, 모든 결과가 나왔다는 뜻이다. 아래 코드로 로딩이 끝나고, 결과값이 준비되었다는 것을 상태 변환
      this.loading = false;
      this.result = true;
      this.scrollToResult();
    },
    async makeResult() {
      // gpt에게 듣는 종 관련 설명
      var gpt_addr =
        "https://gshnajsid6.execute-api.us-east-1.amazonaws.com/dev/" +
        this.breedName;
      var gptResponse = await axios.get(gpt_addr);
      this.breedProp = gptResponse["data"];
    },
    scrollToUpload() {
      // 업로드 창으로 scroll
      this.$refs["fileUpload"].scrollIntoView({
        behavior: "smooth",
        block: "center",
      });
      console.log("middle");
      // upload.scrollIntoView({ behavior: "smooth" });
    },
    scrollToResult() {
      // 결과 창으로 scroll
      if (this.result) {
        this.$refs["analyzeResult"].scrollIntoView({
          behavior: "smooth",
          block: "end",
        });
      }
    },
    downloadImage() {
      const link = document.createElement("a");
      link.href = this.imageUrl;
      link.download = "dogmoji.jpg"; // 다운로드될 파일명
      link.click();
    },
  },
};
</script>

<style scoped>
.empty {
  padding-bottom: 30vh;
}
.box {
  height: 100vh;
  width: 95%;
  max-width: 450px;
  padding: 10px;
  position: absolute;
  top: 50%;
  left: 50%;
  background-image: url("../assets/background.png");
  background-size: cover;
  background-repeat: repeat;
  transform: translate(-50%, -50%);
  border-radius: 5vh;
  filter: drop-shadow(0px 4px 4px rgba(0, 0, 0, 0.25));
  align-items: center;
  overflow-y: scroll;
  overflow-x: hidden;
}

.container {
  top: 200vh;
  left: 50%;
  transform: translate(-50%, -50%);
  position: relative;
}

.drop_box {
  margin: 10px 0;
  padding: 30px;
  height: 50vh;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-direction: column;
  border: 3px dotted #a3a3a3;
  border-radius: 5px;
}
.analyze-container {
  width: 100%;
  display: flex;
  overflow: auto;
  min-height: 100vh;
  align-items: center;
  flex-direction: column;
}
.analyze-analyze {
  gap: 10px;
  width: 100%;
  min-height: 100vh;
  display: flex;
  position: relative;
  align-items: flex-start;
  background-color: #ffffff;
  background-repeat: repeat-y;
  background-size: cover;
}

.real-1 {
  position: absolute;
  max-width: 169px;
  width: 35vw;
  max-height: 169px;
  height: 35vw;
  left: calc(50% + 13vh);
  top: calc(15vh);

  object-fit: cover;
  filter: drop-shadow(0px 4px 4px rgba(0, 0, 0, 0.5));
  border-radius: 89px;
  transform: translate(-50%, -50%);
}
.icon-1 {
  position: absolute;
  width: 25vw;
  max-width: 114px;
  height: 25vw;
  max-height: 114px;
  left: calc(50% + 3vh);
  top: calc(20vh);

  background: #fdffa2;
  object-fit: cover;
  box-shadow: 0px 4px 4px rgba(0, 0, 0, 0.25);
  border-radius: 57px;
  transform: translate(-50%, -50%);
}

.real-2 {
  position: absolute;
  max-width: 170px;
  width: 32vw;
  max-height: 170px;
  height: 32vw;
  left: calc(50% + 10vh);
  top: calc(86vh);

  object-fit: cover;
  filter: drop-shadow(0px 4px 4px rgba(0, 0, 0, 0.25));
  border-radius: 85px;
  transform: translate(-50%, -50%);
}
.icon-2 {
  position: absolute;
  max-width: 100px;
  width: 19vw;
  max-height: 100px;
  height: 19vw;
  left: calc(50% + 18vh);
  top: calc(79vh);

  background: url(../assets/sample_icon2.png), #fdffa2;
  background-size: 70px;
  background-repeat: no-repeat;
  background-position: top;
  box-shadow: 0px 4px 4px rgba(0, 0, 0, 0.25);
  border-radius: 49px;
  object-fit: fill;
  transform: translate(-50%, -50%);
}
.real-3 {
  position: absolute;
  max-width: 141px;
  width: 27vw;
  max-height: 141px;
  height: 27vw;
  left: calc(50% - 15vh);
  top: calc(70vh);

  object-fit: cover;
  object-position: top;
  filter: drop-shadow(0px 4px 4px rgba(0, 0, 0, 0.25));
  border-radius: 61px;
  transform: translate(-50%, -50%);
}
.icon-3 {
  position: absolute;
  max-width: 93px;
  width: 17vw;
  max-height: 93px;
  height: 17vw;
  left: calc(50% - 8vh);
  top: calc(78vh);

  background: url(../assets/sample_icon3.png), #fdffa2;
  background-repeat: no-repeat;
  background-position: top;
  background-size: 80px;
  box-shadow: 0px 4px 4px rgba(0, 0, 0, 0.25);
  border-radius: 46.5px;
  transform: translate(-50%, -50%);
}
.text-group {
  top: 30vh;
  left: 50%;
  display: flex;
  position: absolute;
  transform: translate(-50%, -50%);
  position: relative;
  align-items: center;
  flex-direction: column;
}
.main-text {
  color: rgba(251, 234, 88, 1);
  height: auto;
  position: absolute;
  font-size: 80px;
  font-style: Bold;
  text-align: center;
  font-family: Roboto;
  font-weight: 700;
  line-height: normal;
  font-stretch: normal;
  text-decoration: none;
  text-shadow: 1px 2px 2px #888;
}
.main-text2 {
  color: rgba(0, 0, 0, 1);
  height: auto;
  position: absolute;
  top: 15vh;
  font-size: 28px;
  font-style: Bold;
  text-align: center;
  font-family: Roboto;
  font-weight: 700;
  line-height: normal;
  font-stretch: normal;
  text-decoration: none;
  text-shadow: 1px 2px 2px #bbb;
}

.button {
  position: absolute;
  top: 25vh;
  filter: drop-shadow(1px 2px 2px #bbb);
}

.result {
  padding-top: 30vh;
  top: 2400px;
  left: 50%;
  transform: translate(-50%, -50%);
  position: relative;
  padding-bottom: 30vh;
}

.input-image {
  align-items: center;
  justify-content: center;
  flex-direction: column;
}

.form {
  width: 75vw;
  max-width: 400px;
}
</style>
