<template>
  <div class="main" @dragenter.prevent @dragover.prevent @drop.prevent="onDrop">
    <progress v-if="uploadProgress !== null" :value="uploadProgress" max="100"></progress>
    <UploadPopup v-model="showUploadPopup" @upload="onUploadClicked" @createFolder="createFolder"></UploadPopup>
    <button class="upload-button circle" @click="showUploadPopup = true">
      <img
        style="filter: invert(100%)"
        src="https://cdnjs.cloudflare.com/ajax/libs/material-design-icons/4.0.0/png/file/upload_file/materialicons/36dp/2x/baseline_upload_file_black_36dp.png"
        alt="Upload"
        width="36"
        height="36"
        @contextmenu.prevent
      />
    </button>
    <div class="app-bar">
      <input type="search" v-model="search" aria-label="Search" />
      <div class="menu-button">
        <button class="circle" @click="showMenu = true">
          <svg
            xmlns="http://www.w3.org/2000/svg"
            viewBox="0 0 448 512"
            width="24"
            height="24"
            title="Menu"
            style="display: block; margin: 4px"
          >
            <path
              d="M120 256c0 30.9-25.1 56-56 56s-56-25.1-56-56s25.1-56 56-56s56 25.1 56 56zm160 0c0 30.9-25.1 56-56 56s-56-25.1-56-56s25.1-56 56-56zm104 56c-30.9 0-56-25.1-56-56s25.1-56 56-56s56 25.1 56 56z"
            />
          </svg>
        </button>
        <Menu
          v-model="showMenu"
          :items="[
            { text: '名称A-Z' },
            { text: '大小↑' },
            { text: '大小↓' },
            { text: '粘贴' },
            { text: '关于 LI Innovation 共享存储库' }  <!-- 添加新项 -->
          ]"
          @click="onMenuClick"
        />
      </div>
    </div>
    <ul class="file-list">
      <!-- 省略其余代码 -->
    </ul>
    <div v-if="loading" style="margin-top: 12px; text-align: center">
      <span>加载中...</span>
    </div>
    <div v-else-if="!filteredFiles.length && !filteredFolders.length" style="margin-top: 12px; text-align: center">
      <span>没有文件</span>
    </div>
    <Dialog v-model="showContextMenu">
      <div v-text="focusedItem.key || focusedItem" class="contextmenu-filename" @click.stop.prevent></div>
      <ul v-if="typeof focusedItem === 'string'" class="contextmenu-list">
        <li>
          <button @click="copyLink(`/?p=${encodeURIComponent(focusedItem)}`)">
            <span>复制链接</span>
          </button>
        </li>
        <li>
          <button style="color: red" @click="removeFile(focusedItem + '_$folder$')">
            <span>删除</span>
          </button>
        </li>
      </ul>
      <ul v-else class="contextmenu-list">
        <li>
          <button @click="renameFile(focusedItem.key)">
            <span>重命名</span>
          </button>
        </li>
        <li>
          <a :href="`/raw/${focusedItem.key}`" target="_blank" download>
            <span>下载</span>
          </a>
        </li>
        <li>
          <button @click="clipboard = focusedItem.key">
            <span>复制</span>
          </button>
        </li>
        <li>
          <button @click="copyLink(`/raw/${focusedItem.key}`)">
            <span>复制链接</span>
          </button>
        </li>
        <li>
          <button style="color: red" @click="removeFile(focusedItem.key)">
            <span>删除</span>
          </button>
        </li>
      </ul>
    </Dialog>
  </div>
</template>

<script>
import {
  generateThumbnail,
  blobDigest,
  multipartUpload,
  SIZE_LIMIT,
} from "/assets/main.mjs";
import Dialog from "./Dialog.vue";
import Menu from "./Menu.vue";
import MimeIcon from "./MimeIcon.vue";
import UploadPopup from "./UploadPopup.vue";

export default {
  data: () => ({
    cwd: new URL(window.location).searchParams.get("p") || "",
    files: [],
    folders: [],
    clipboard: null,
    focusedItem: null,
    loading: false,
    order: null,
    search: "",
    showContextMenu: false,
    showMenu: false,
    showUploadPopup: false,
    uploadProgress: null,
    uploadQueue: [],
  }),

  computed: {
    filteredFiles() {
      let files = this.files;
      if (this.search) {
        files = files.filter((file) =>
          file.key.split("/").pop().includes(this.search)
        );
      }
      return files;
    },

    filteredFolders() {
      let folders = this.folders;
      if (this.search) {
        folders = folders.filter((folder) => folder.includes(this.search));
      }
      return folders;
    },
  },

  methods: {
    copyLink(link) {
      const url = new URL(link, window.location.origin);
      navigator.clipboard.writeText(url.toString());
    },

    async copyPaste(source, target) {
      const uploadUrl = `/api/write/items/${target}`;
      await axios.put(uploadUrl, "", {
        headers: { "x-amz-copy-source": encodeURIComponent(source) },
      });
    },

    async createFolder() {
      try {
        const folderName = window.prompt("请输入文件夹名称");
        if (!folderName) return;
        this.showUploadPopup = false;
        const uploadUrl = `/api/write/items/${this.cwd}${folderName}/_$folder$`;
        await axios.put(uploadUrl, "");
        this.fetchFiles();
      } catch (error) {
        fetch("/api/write/")
          .then((value) => {
            if (value.redirected) window.location.href = value.url;
          })
          .catch(() => {});
        console.log(`Create folder failed`);
      }
    },

    fetchFiles() {
      this.files = [];
      this.folders = [];
      this.loading = true;
      fetch(`/api/children/${this.cwd}`)
        .then((res) => res.json())
        .then((files) => {
          this.files = files.value;
          if (this.order) {
            this.files.sort((a, b) => {
              if (this.order === "size") {
                return b.size - a.size;
              }
            });
          }
          this.folders = files.folders;
          this.loading = false;
        });
    },

    formatSize(size) {
      const units = ["B", "KB", "MB", "GB", "TB"];
      let i = 0;
      while (size >= 1024) {
        size /= 1024;
        i++;
      }
      return `${size.toFixed(1)} ${units[i]}`;
    },

    onDrop(ev) {
      let files;
      if (ev.dataTransfer.items) {
        files = [...ev.dataTransfer.items]
          .filter((item) => item.kind === "file")
          .map((item) => item.getAsFile());
      } else files = ev.dataTransfer.files;
      this.uploadFiles(files);
    },

    onMenuClick(text) {
      switch (text) {
        case "名称A-Z":
          this.order = null;
          break;
        case "大小↑":
          this.order = "大小↑";
          break;
        case "大小↓":
          this.order = "大小↓";
          break;
        case "粘贴":
          return this.pasteFile();
        case "关于 LI Innovation 共享存储库":  // 添加处理逻辑
          window.open("https://liiproject.org/liidb", "_blank");
          break;
      }
      this.files.sort((a, b) => {
        if (this.order === "大小↑") {
          return a.size - b.size;
        } else if (this.order === "大小↓") {
          return b.size - a.size;
        } else {
          return a.key.localeCompare(b.key);
        }
      });
    },

    onUploadClicked(fileElement) {
      if (!fileElement.value) return;
      this.uploadFiles(fileElement.files);
    },

    pasteFile() {
      const file = this.clipboard;
      if (typeof file === "string") {
        return this.copyPaste(file, this.cwd + file.split("/").pop());
      }
      return this.uploadFiles([file]);
    },

    async removeFile(file) {
      try {
        if (window.confirm(`Are you sure you want to delete ${file}?`)) {
          await axios.delete(`/api/write/items/${file}`);
          this.fetchFiles();
        }
      } catch (error) {
        console.log(`Delete ${file} failed`, error);
      }
    },

    renameFile(file) {
      const dest = window.prompt("重命名文件为", file);
      if (!dest) return;
      const prefix = file.split("/").slice(0, -1).join("/") + "/";
      return this.copyPaste(file, prefix + dest).then(() =>
        this.removeFile(file)
      );
    },

    async uploadFiles(files) {
      if (!files.length) return;
      this.uploadQueue = [...this.uploadQueue, ...files];
      const next = async () => {
        const file = this.uploadQueue.shift();
        if (!file) return this.fetchFiles();
        try {
          this.uploadProgress = 0;
          await multipartUpload(
            file,
            `${this.cwd}${file.webkitRelativePath || file.name}`,
            (progress) => (this.uploadProgress = progress)
          );
          this.uploadProgress = null;
        } catch (error) {
          fetch("/api/write/")
            .then((value) => {
              if (value.redirected) window.location.href = value.url;
            })
            .catch(() => {});
          console.log(`Upload ${file.name} failed`, error);
        }
        await next();
      };
      next();
    },
  },

  mounted() {
    this.fetchFiles();
  },

  components: {
    Dialog,
    Menu,
    MimeIcon,
    UploadPopup,
  },
};
</script>

