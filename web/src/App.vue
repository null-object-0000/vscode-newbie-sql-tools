<template>
  <div class="app-container">
    <a-tabs v-model:active-key="activeTab">
      <a-tab-pane
        key="results"
        :title="
          pageData.platform ? `【${pageData.platform}】Results` : 'Results'
        "
      >
        <div
          style="margin-bottom: 8px; margin-left: 16px"
          v-if="pageData.table.loading === false"
        >
          <a-form v-model="pageData.table.config" layout="inline">
            <a-button style="margin-right: 20px" size="small" disabled>
              <template #icon>
                <icon-refresh />
              </template>
              刷新
            </a-button>
            <a-button style="margin-right: 20px" size="small" disabled>
              <template #icon>
                <icon-download />
              </template>
              下载
            </a-button>
          </a-form>
        </div>

        <div
          ref="listTableRef"
          style="width: 100%; height: calc(100vh - 100px)"
        ></div>
      </a-tab-pane>
      <a-tab-pane key="logs" title="Logs">
        <a-timeline :reverse="true">
          <a-timeline-item
            v-for="log of pageData.logs"
            :dotColor="levelColorMap[log.level]"
            :label="`【${log.level}】${log.message}`"
            >{{ log.time }}</a-timeline-item
          >
        </a-timeline>
      </a-tab-pane>
      <a-tab-pane key="sql" title="SQL">
        <pre :class="themeMode" v-html="pageData.highlightedSQL"></pre>
      </a-tab-pane>
      <a-tab-pane key="chart" title="Chart">
        <pre>{{ pageData.sql }}</pre>
      </a-tab-pane>
      <a-tab-pane key="config" title="Config">
        <pre>{{ pageData.sql }}</pre>
      </a-tab-pane>
    </a-tabs>
  </div>
</template>

<script setup>
import {
  watch,
  ref,
  onBeforeMount,
  reactive,
  onMounted,
  shallowRef,
} from "vue";
import * as VTable from "@visactor/vtable";
import hljs from "highlight.js/lib/core";
import sql from "highlight.js/lib/languages/sql";

// Then register the languages you need
hljs.registerLanguage("sql", sql);

const pageData = reactive({
  eventMessage: {
    command: "",
    data: null,
  },
  sql: "",
  highlightedSQL: "",
  logs: [],
  table: {
    loading: false,
    platform: null,
  },
});

const levelColorMap = {
  INFO: "blue",
  WARN: "orange",
  ERROR: "red",
};

const listTableRef = ref();
const tableInstance = shallowRef();

const updateColumns = () => {
  const columns = pageData.eventMessage.data.columns.map((item) => {
    return {
      title: item.title || item.name,
      field: item.name,
      width: "auto",
    };
  });

  pageData.table.columns = columns;

  tableInstance.value.updateColumns(columns);

  return columns;
};

const activeTab = ref("logs");

window.onmessage = (event) => {
  pageData.eventMessage = event.data;
  console.log("onmessage", pageData.eventMessage);

  if (pageData.eventMessage.command === "begin") {
    pageData.table.loading = true;
    pageData.sql = pageData.eventMessage.data.sql;
    pageData.platform = pageData.eventMessage.data.platform;

    pageData.logs = [];

    tableInstance.value.setRecords([]);

    activeTab.value = "logs";
  } else if (pageData.eventMessage.command === "result") {
    pageData.table.loading = false;

    updateColumns();
    tableInstance.value.setRecords(pageData.eventMessage.data.rows);

    activeTab.value = "results";
  } else if (pageData.eventMessage.command === "failed") {
    pageData.table.loading = false;
    pageData.sql = pageData.eventMessage.data.sql;

    activeTab.value = "logs";
  } else if (pageData.eventMessage.command === "theme") {
    // light | dark
    themeMode.value = pageData.eventMessage.data.theme;
  }

  if (pageData.eventMessage.log) {
    pageData.logs.push(pageData.eventMessage.log);
  }

  if (pageData.sql) {
    pageData.highlightedSQL = hljs.highlight(pageData.sql, {
      language: "sql",
    }).value;
  }
};

/**
 * light | dark
 */
const themeMode = ref(window.themeMode || "dark");
const updateTheme = (theme) => {
  if (theme === "light") {
    document.body.setAttribute("arco-theme", "light");
    tableInstance.value.updateTheme(VTable.themes.LIGHT);
  } else {
    document.body.setAttribute("arco-theme", "dark");
    tableInstance.value.updateTheme(VTable.themes.DARK);
  }
};

watch(themeMode, (value) => {
  updateTheme(value);
});

onMounted(() => {
  const option = {
    theme: VTable.themes.DARK,
    rowSeriesNumber: true,
    autoFillWidth: true,
    autoWrapText: true,
    tooltip: {
      renderMode: "html",
      isShowOverflowTextTooltip: true,
      overflowTextTooltipDisappearDelay: 1000,
      confine: false,
    },
    widthMode: "autoWidth",
    heightMode: "autoHeight",
    keyboardOptions: {
      moveEditCellOnArrowKeys: true,
      copySelected: true,
      pasteValueToCell: true,
    },
    editor: "",
    emptyTip: {
      text: "暂无数据",
      textStyle: {
        color: themeMode.value === "light" ? "#000" : "#fff",
      },
    },
  };

  // 创建 VTable 实例

  tableInstance.value = new VTable.ListTable(listTableRef.value, option);

  updateTheme(themeMode.value);
});
</script>

<style>
body {
  background-color: var(--color-bg-1);
}

pre.dark {
  pre code.hljs {
    display: block;
    overflow-x: auto;
    padding: 1em;
  }
  code.hljs {
    padding: 3px 5px;
  }
  .hljs {
    color: #abb2bf;
    background: #282c34;
  }
  .hljs-comment,
  .hljs-quote {
    color: #5c6370;
    font-style: italic;
  }
  .hljs-doctag,
  .hljs-formula,
  .hljs-keyword {
    color: #c678dd;
  }
  .hljs-deletion,
  .hljs-name,
  .hljs-section,
  .hljs-selector-tag,
  .hljs-subst {
    color: #e06c75;
  }
  .hljs-literal {
    color: #56b6c2;
  }
  .hljs-addition,
  .hljs-attribute,
  .hljs-meta .hljs-string,
  .hljs-regexp,
  .hljs-string {
    color: #98c379;
  }
  .hljs-attr,
  .hljs-number,
  .hljs-selector-attr,
  .hljs-selector-class,
  .hljs-selector-pseudo,
  .hljs-template-variable,
  .hljs-type,
  .hljs-variable {
    color: #d19a66;
  }
  .hljs-bullet,
  .hljs-link,
  .hljs-meta,
  .hljs-selector-id,
  .hljs-symbol,
  .hljs-title {
    color: #61aeee;
  }
  .hljs-built_in,
  .hljs-class .hljs-title,
  .hljs-title.class_ {
    color: #e6c07b;
  }
  .hljs-emphasis {
    font-style: italic;
  }
  .hljs-strong {
    font-weight: 700;
  }
  .hljs-link {
    text-decoration: underline;
  }
}

pre.light {
  pre code.hljs {
    display: block;
    overflow-x: auto;
    padding: 1em;
  }
  code.hljs {
    padding: 3px 5px;
  }
  .hljs {
    color: #383a42;
    background: #fafafa;
  }
  .hljs-comment,
  .hljs-quote {
    color: #a0a1a7;
    font-style: italic;
  }
  .hljs-doctag,
  .hljs-formula,
  .hljs-keyword {
    color: #a626a4;
  }
  .hljs-deletion,
  .hljs-name,
  .hljs-section,
  .hljs-selector-tag,
  .hljs-subst {
    color: #e45649;
  }
  .hljs-literal {
    color: #0184bb;
  }
  .hljs-addition,
  .hljs-attribute,
  .hljs-meta .hljs-string,
  .hljs-regexp,
  .hljs-string {
    color: #50a14f;
  }
  .hljs-attr,
  .hljs-number,
  .hljs-selector-attr,
  .hljs-selector-class,
  .hljs-selector-pseudo,
  .hljs-template-variable,
  .hljs-type,
  .hljs-variable {
    color: #986801;
  }
  .hljs-bullet,
  .hljs-link,
  .hljs-meta,
  .hljs-selector-id,
  .hljs-symbol,
  .hljs-title {
    color: #4078f2;
  }
  .hljs-built_in,
  .hljs-class .hljs-title,
  .hljs-title.class_ {
    color: #c18401;
  }
  .hljs-emphasis {
    font-style: italic;
  }
  .hljs-strong {
    font-weight: 700;
  }
  .hljs-link {
    text-decoration: underline;
  }
}
</style>
