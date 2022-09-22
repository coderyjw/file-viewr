<template>
  <div :class="{ hidden }">
    <div class="banner">
      <div class="container">
        <h1>
          <div class="file-select">
            <button type="button" style="margin-right: 20px" @click.stop="input = !input">
              【点击切换】{{ input ? "🐧 输入网址" : "👆🏻 上传预览" }}
            </button>
            <button type="button" @click="overlay = !overlay">{{ overlay ? "🏄‍隐藏浮层" : "👁显示浮层" }}</button>
          </div>
          <div class="overlay" v-if="overlay">
            <input v-if="input" type="text" v-model="url" placeholder="http://" />
            <input v-else type="file" @change="handleChange" />
            <button type="button" v-if="input" @click.stop="loadFromUrl">预览</button>
            <div class="upload-cover" v-else>📃 {{ filename || "请选择文件上传" }}</div>
          </div>
          <a href="/">Vue在线文档查看器</a>
        </h1>
      </div>
    </div>
    <div class="container">
      <div v-show="loading" class="well loading">正在加载中，请耐心等待...</div>
      <div v-show="!loading" class="well" ref="output"></div>
    </div>
  </div>
</template>

<script>
import { getExtend, readBuffer, render } from "@/components/util";
import { parse } from "qs";
import axios from "axios";

/**
 * 支持嵌入式显示，基于postMessage支持跨域
 * 示例代码：
 *
 */
export default {
  name: "HelloWorld",
  props: {
    msg: String,
  },
  data() {
    return {
      // 加载状态跟踪
      loading: false,
      // 上个渲染实例
      last: null,
      // 隐藏头部，当基于消息机制渲染，将隐藏
      hidden: false,
      // 使用输入框
      input: false,
      // 浮层显示
      overlay: true,
      // 文件名
      filename: "",
      // 网址
      url: "http://flyfish.group/%E6%95%B0%E6%8D%AE%E4%B8%AD%E5%8F%B0%E7%AC%94%E8%AE%B0(1).docx",
    };
  },
  created() {
    // 允许使用预留的消息机制发送二进制数据，必须在url后添加?name=xxx.xxx&from=xxx
    const { from, name } = parse(location.search.substr(1));
    if (from) {
      window.addEventListener("message", (event) => {
        const { origin, data: blob } = event;
        if (origin === from && blob instanceof Blob) {
          // 构造响应，自动渲染
          const file = new File([blob], name, {});
          this.hidden = true;
          this.handleChange({ target: { files: [file] } });
        }
      });
    }
  },
  methods: {
    // 从url加载
    loadFromUrl() {
      this.loading = true;
      // 要预览的文件地址
      const { url } = this;
      const filename = url.substr(url.lastIndexOf("/") + 1);
      // 拼接iframe请求url
      axios({
        url,
        method: "get",
        responseType: "blob",
      })
        .then(({ data }) => {
          if (!data) {
            console.error("文件下载失败");
          }
          const file = new File([data], filename, {});
          this.handleChange({ target: { files: [file] } });
        })
        .finally(() => {
          this.loading = false;
        });
    },
    async handleChange(e) {
      this.loading = true;
      try {
        const [file] = e.target.files;
        this.filename = (file.name && decodeURIComponent(file.name)) || "";
        const arrayBuffer = await readBuffer(file);
        this.loading = false;
        this.last = await this.displayResult(arrayBuffer, file);
      } catch (e) {
        console.error(e);
      } finally {
        this.loading = false;
      }
    },
    displayResult(buffer, file) {
      // 取得文件名
      const { name } = file;
      // 取得扩展名
      const extend = getExtend(name);
      // 输出目的地
      const { output } = this.$refs;
      // 生成新的dom
      const node = document.createElement("div");
      // 添加孩子，防止vue实例替换dom元素
      if (this.last) {
        console.log("last", this.last, this.last.$destroy);
        output.removeChild(this.last.$el);
        this.last.$destroy();
      }
      const child = output.appendChild(node);
      // 调用渲染方法进行渲染
      return new Promise((resolve, reject) => render(buffer, extend, child).then(resolve).catch(reject));
    },
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.banner {
  overflow: auto;
  text-align: center;
  background-color: #12b6ff;
  color: #fff;
}

.hidden .banner {
  display: none;
}

.hidden .well {
  height: calc(100vh - 12px);
}

.overlay {
  position: absolute;
  transition: all;
  z-index: 1000;
  opacity: 0.4;
  top: 50px;
  left: 112px;
  padding: 20px;
  border-radius: 5px;
  background: white;
  border: 1px solid silver;
}

.overlay:hover {
  opacity: 1;
}

.file-select {
  position: absolute;
  left: 5%;
  line-height: 35px;
  margin-left: 20px;
}

.banner a {
  color: #fff;
  text-decoration: none;
}

.banner h1 {
  font-size: 20px;
  line-height: 2;
  margin: 0.5em 0;
}

.well {
  display: block;
  background-color: #f2f2f2;
  border: 1px solid #ccc;
  margin: 5px;
  width: calc(100% - 12px);
  height: calc(100vh - 73px);
  overflow: auto;
}

.loading {
  text-align: center;
  padding-top: 50px;
}

.file-select button {
  background: #fafafa;
}

.overlay button {
  background: #12b6ff;
  color: white;
}

button {
  outline: none;
  border-radius: 20px;
  border: 1px solid #e3e3e3;
  line-height: 19px;
  padding: 5px 12px;
  cursor: pointer;
}

.overlay input[type="text"] {
  line-height: 19px;
  height: 30px;
  width: 300px;
  outline: none;
  border: 1px solid silver;
  border-radius: 6px;
  margin-right: 10px;
}

.overlay input[type="file"] {
  position: absolute;
  opacity: 0;
  width: 100%;
  height: 100%;
  left: 0;
  top: 0;
  z-index: 2;
  cursor: pointer;
}

.upload-cover {
  z-index: 1;
  pointer-events: none;
  color: black;
}

.messages .warning {
  color: #cc6600;
}
</style>

<style>
.pptx-wrapper {
  max-width: 1000px;
  margin: 0 auto;
}
</style>
