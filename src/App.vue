<template>
  <el-row class="row-body">
    <el-col :span="4" :offset="1" class="col">
      <div class="domain-input-wrapper">
        <span>课程名:</span>
        <el-input
          placeholder="输入课程名"
          v-model="classInput"
          class="domain-input"
        >
        </el-input>
      </div>
      <div style="margin-top: 20px">
        输入主题列表或
        <el-upload
          ref="upload-topic-names"
          action=""
          accept=".txt"
          :auto-upload="false"
          :on-change="onUploadChange"
          :on-remove="onUploadRemove"
          :limit="1"
        >
          <el-button class="mr-2" plain size="small" type="primary">
            <i class="el-icon-upload"></i>上传文件
          </el-button>
        </el-upload>
      </div>
      <el-input
        type="textarea"
        placeholder="输入主题列表，回车隔开"
        v-model="topicInput"
        resize="none"
        :rows="20"
        class="topic-names-input"
      >
      </el-input>
      <div class="btn-group">
        <el-button type="success" @click="onCheckStatusClick"
          >查询状态</el-button
        >
        <el-button type="primary" @click="obtainData">获取数据</el-button>
        <el-button type="primary" @click="handleFacetClick">分面抽取</el-button>
      </div>
    </el-col>

    <el-col :span="16" :offset="2" class="col">
      <el-card shadow="always" class="status">
        <div v-if="className.length">
          课程: {{ className }}
          <el-divider direction="vertical"></el-divider>
          {{ topicNames.length }}个主题
        </div>
        <div>
          当前状态: {{ constructStatus }}
          <el-progress
            v-if="percent"
            :text-inside="true"
            :stroke-width="24"
            :percentage="percent"
            status="success"
          ></el-progress>
        </div>
      </el-card>

      <el-tabs v-model="activeTab" type="card" class="result-panel-tab">
        <el-tab-pane label="初始分面集" name="initFacets">
          <el-cascader-panel
            v-loading="!initFacets"
            :options="initFacetOptions"
          ></el-cascader-panel>
        </el-tab-pane>
        <el-tab-pane label="最终分面集" name="Facets">
          <el-cascader-panel
            v-loading="!facetSet"
            :options="facetSetOptions"
          ></el-cascader-panel>
          <el-button
            v-if="facetSet"
            type="primary"
            icon="el-icon-download"
            @click="downloadFacetSet"
            >下载</el-button
          >
        </el-tab-pane>
      </el-tabs>
      <a style="display: none" ref="_download" target="_blank"></a>
    </el-col>
  </el-row>
</template>

<script>
import axios from "axios";

export default {
  name: "App",
  components: {},
  data() {
    return {
      classInput: "",
      topicInput: "",
      className: "",
      topicNames: [],
      constructStatus: "就绪",
      percent: null,
      initFacets: null,
      facetSet: null,
      activeTab: "initFacets",
    };
  },
  methods: {
    async onCheckStatusClick() {
      this.percent = null;
      this.initFacets = null;
      this.facetSet = null;
      this.className = this.classInput.trim();
      this.topicNames = this.topicInput
        .split(/\n/)
        .map((t) => t.trim())
        .filter((v) => v.length !== 0);
      const crawlerResponse = await axios.post(
        "http://10.181.60.186:5678/crawl-status",
        {
          className: this.className,
          topicNames: this.topicNames,
        }
      );
      const crawlerStatus = crawlerResponse.data;
      if (crawlerStatus.status === "NOT_EXIST") {
        this.constructStatus = "数据缺失";
      } else if (crawlerStatus.status === "NOT_COMPLETE") {
        this.constructStatus = "数据不完整";
      } else if (crawlerStatus.status === "CONSTRUCTING") {
        this.constructStatus = "数据初始化中";
        this.percent =
          (1 -
            crawlerStatus.waitingForConstruct.length / this.topicNames.length) *
          100;
      } else if (crawlerStatus.status === "EXIST") {
        const feResponse = await axios.post(
          "http://10.181.60.186:4675/facet-extract-status",
          {
            className: this.className,
            topicNames: this.topicNames,
          }
        );
        const feStatus = feResponse.data;
        if (feStatus.status === "NOT_EXIST") {
          this.constructStatus = "数据采集完成，未构建分面集";
        } else if (feStatus.status === "NOT_COMPLETE") {
          this.constructStatus = "数据采集完成，分面集不完整";
        } else if (feStatus.status === "EXIST") {
          this.constructStatus = "已构建分面集";
          this.initFacets = (
            await axios.post("http://10.181.60.186:4675/get-init-facets", {
              className: this.className,
              topicNames: this.topicNames,
            })
          ).data;
          this.facetSet = (
            await axios.post("http://10.181.60.186:4675/get-facet-set", {
              className: this.className,
              topicNames: this.topicNames,
            })
          ).data;
        } else if (feStatus.status === "CONSTRUCTING") {
          const meta = feStatus.meta.step;
          switch (meta) {
            case "PREPROCESS_ANALYSE_INIT_FACETS":
              this.constructStatus = "解析初始分面";
              break;
            case "PREPROCESS_ANALYSE_SUMMARY_EMBEDDING":
              this.constructStatus = "计算主题Embedding";
              break;
            case "PREPROCESS_ANALYSE_TOPIC_HIERARY":
              this.constructStatus = "解析主题上下位关系";
              break;
            case "ALG_CALCULATE_SIMILARITIES":
              this.constructStatus = "分析主题相似度";
              break;
            case "ALG_TOPIC_CLUSTERING":
              this.constructStatus = "主题聚类";
              break;
            case "ALG_GENERATE_PROPAGATION_MATRIX":
              this.constructStatus = "计算传播矩阵";
              break;
            case "ALG_LABEL_PROPAGATION":
              this.constructStatus = "执行标签传播算法";
              break;
            case "ALG_ANALYSE_FACET_SET":
              this.constructStatus = "解析分面集";
              break;
          }
        } else {
          this.constructStatus = "数据采集完成，未构建分面集";
        }
      }
    },

    async handleFacetClick(e) {
      this.constructStatus = "分面抽取";
      this.percent = null;
      this.initFacets = null;
      this.facetSet = null;
      this.className = this.classInput.trim();
      this.topicNames = this.topicInput
        .split(/\n/)
        .map((t) => t.trim())
        .filter((v) => v.length !== 0);
      const crawlerResponse = await axios.post(
        "http://10.181.60.186:5678/crawl-status",
        {
          className: this.className,
          topicNames: this.topicNames,
        }
      );
      const crawlerStatus = crawlerResponse.data;
      if (crawlerStatus.status === "EXIST") {
        axios.post("http://10.181.60.186:4675/facet-extract", {
          className: this.className,
          topicNames: this.topicNames,
        });
        this.$message({
          message: "开始构建分面集",
          center: true,
          type: "success",
        });
      } else {
        this.$message({
          message: "数据不完整，请先获取数据",
          center: true,
          type: "error",
        });
      }
    },

    async obtainData() {
      this.constructStatus = "获取数据";
      this.percent = null;
      this.initFacets = null;
      this.facetSet = null;
      this.className = this.classInput.trim();
      this.topicNames = this.topicInput
        .split(/\n/)
        .map((t) => t.trim())
        .filter((v) => v.length !== 0);
      const crawlerResponse = await axios.post(
        "http://10.181.60.186:5678/crawl-status",
        {
          className: this.className,
          topicNames: this.topicNames,
        }
      );
      const crawlerStatus = crawlerResponse.data;
      if (crawlerStatus.status !== "EXIST") {
        axios.post("http://10.181.60.186:5678/crawl", {
          className: this.className,
          topicNames: this.topicNames,
        });
      }
      this.$message({
        message: "开始获取数据",
        center: true,
        type: "success",
      });
    },

    downloadFacetSet() {
      axios
        .post(
          "http://10.181.60.186:4675/download-facet-set",
          {
            className: this.className,
            topicNames: this.topicNames,
          },
          {
            responseType: "blob",
          }
        )
        .then((res) => {
          console.log(res);
          if (res) {
            const url = URL.createObjectURL(
              new Blob([res.data], { type: "application/x-zip-compressed" })
            );
            this.$refs._download.href = url;
            this.$refs._download.setAttribute(
              "download",
              `${this.className}_facet_set.zip`
            );
            this.$refs._download.click();
          }
        });
    },
    onUploadChange(e) {
      const f = e.raw;
      let reader = new FileReader();
      reader.readAsText(f);
      reader.onloadend = () => {
        if (!reader.error) {
          this.topicInput = reader.result;
        } else {
          this.$message({
            message: "上传出错",
            center: true,
            type: "error",
          });
        }
      };
    },
    onUploadRemove(e) {
      this.topicInput = "";
    },
  },

  computed: {
    initFacetOptions() {
      let options = [];
      if (!this.initFacets) {
        return options;
      }
      for (let k in this.initFacets) {
        options.push({
          value: k,
          label: k,
          children: this.initFacets[k].map((fs) => {
            return {
              value: fs,
              label: fs,
            };
          }),
        });
      }
      return options;
    },

    facetSetOptions() {
      let options = [];
      if (!this.facetSet) {
        return options;
      }
      for (let k in this.facetSet) {
        options.push({
          value: k,
          label: k,
          children: this.facetSet[k].map((fs) => {
            return {
              value: fs,
              label: fs,
            };
          }),
        });
      }
      return options;
    },
  },
};
</script>

<style scoped>
.row-body {
  margin-top: 10vh;
  height: 90vh;
  max-height: 90vh;
}

.col {
  display: flex;
  flex-direction: column;
  height: 100%;
}

.domain-input-wrapper {
  display: flex;
  align-items: center;
}

.domain-input {
  margin-left: 20px;
  flex-basis: 0;
  flex-grow: 1;
}

.topic-names-input {
  margin-bottom: 2vh;
}

.btn-group {
  display: flex;
  justify-content: space-evenly;
}

.status {
  line-height: 2em;
}

.result-panel-tab {
  margin-top: 20px;
}

.result-panel {
  height: 45vh;
}
</style>
