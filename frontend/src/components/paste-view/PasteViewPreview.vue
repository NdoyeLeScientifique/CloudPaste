<script setup>
// PasteViewPreview组件 - 用于渲染Markdown内容并提供预览功能
// 该组件使用Vditor库渲染复杂的Markdown内容，支持代码高亮、数学公式等高级特性
import { ref, onMounted, watch, nextTick } from "vue";
import Vditor from "vditor";
import "vditor/dist/index.css"; // 引入Vditor样式
import { debugLog } from "./PasteViewUtils";

// 定义组件接收的属性
const props = defineProps({
  // 是否为暗色模式，控制渲染主题
  darkMode: {
    type: Boolean,
    required: true,
  },
  // 要渲染的Markdown内容
  content: {
    type: String,
    default: "",
  },
  // 是否为开发环境
  isDev: {
    type: Boolean,
    default: false,
  },
  // 是否启用调试日志
  enableDebug: {
    type: Boolean,
    default: false,
  },
});

// 定义组件可触发的事件
const emit = defineEmits(["rendered"]);
// 用于保存预览元素的引用
const previewElement = ref(null);
// 跟踪内容是否已经渲染
const contentRendered = ref(false);

// 监听暗色模式变化，当主题改变时重新渲染内容
watch(
    () => props.darkMode,
    () => {
      if (props.content) {
        // 暗色模式变化时重新渲染
        nextTick(() => {
          renderContent(props.content);
        });
      }
    }
);

// 监听内容变化，当内容改变时重新渲染
watch(
    () => props.content,
    (newContent) => {
      if (newContent) {
        contentRendered.value = false;
        nextTick(() => {
          renderContent(newContent);
        });
      }
    }
);

// 渲染内容的方法，处理DOM可用性和兼容性问题
const renderContent = (content) => {
  if (!content) {
    console.warn("没有内容可渲染");
    return;
  }

  // 延迟检查预览元素，确保DOM已挂载
  if (!previewElement.value) {
    debugLog(props.enableDebug, props.isDev, "预览元素尚未准备好，将在下一个渲染周期尝试");
    // 使用nextTick等待DOM更新后再尝试渲染
    nextTick(() => {
      if (previewElement.value) {
        debugLog(props.enableDebug, props.isDev, "预览元素已就绪，现在开始渲染");
        renderContentInternal(content);
      } else {
        console.error("预览元素始终不可用，无法渲染内容");
      }
    });
    return;
  }

  renderContentInternal(content);
};

// 内部渲染实现，使用Vditor的API渲染Markdown
const renderContentInternal = (content) => {
  // 清空之前的内容，避免重复渲染
  if (previewElement.value) {
    // 完全清除之前的内容和样式
    previewElement.value.innerHTML = "";
    // 移除可能残留的主题相关类
    previewElement.value.classList.remove("vditor-reset--dark", "vditor-reset--light");

    try {
      // 添加一个小延迟确保DOM更新完成
      setTimeout(() => {
        // 使用 Vditor 的预览 API 渲染内容
        // 配置了主题、代码高亮、数学公式等渲染选项
        Vditor.preview(previewElement.value, content, {
          mode: "dark-light", // 支持明暗主题
          theme: {
            current: props.darkMode ? "dark" : "light", // 根据darkMode设置主题
          },
          cdn: "https://cdn.jsdelivr.net/npm/vditor@3.10.9", // 添加CDN配置，确保资源正确加载
          hljs: {
            lineNumber: true, // 代码块显示行号
            style: props.darkMode ? "vs2015" : "github", // 代码高亮样式
          },
          markdown: {
            toc: true, // 启用目录
            mark: true, // 启用标记
            footnotes: true, // 启用脚注
            autoSpace: true, // 自动空格
            media: true, // 启用媒体链接解析（视频、音频等）
            listStyle: true, // 启用列表样式支持
            // 添加任务列表支持
            task: true, // 启用任务列表
            // 图表渲染相关配置
            mermaid: {
              theme: "default", // 使用固定的主题，不跟随暗色模式变化
              useMaxWidth: false, // 不使用最大宽度限制
            },
            flowchart: {
              theme: "default", // 使用固定的主题
            },
            // 固定图表样式
            fixDiagramTheme: true, // 自定义属性，用于CSS选择器中识别
          },
          math: {
            engine: "KaTeX", // 数学公式渲染引擎
            inlineDigit: true, // 启用行内数学公式
          },
          after: () => {
            // 渲染完成后的回调
            debugLog(props.enableDebug, props.isDev, "Markdown 内容渲染完成");

            // 强制添加对应主题的类
            if (props.darkMode) {
              previewElement.value.classList.add("vditor-reset--dark");
              previewElement.value.classList.remove("vditor-reset--light");
            } else {
              previewElement.value.classList.add("vditor-reset--light");
              previewElement.value.classList.remove("vditor-reset--dark");
            }

            // 为所有图表容器添加固定样式类
            const diagramContainers = previewElement.value.querySelectorAll(".language-mermaid, .language-flow, .language-plantuml, .language-gantt");
            diagramContainers.forEach((container) => {
              container.classList.add("diagram-fixed-theme");
            });

            // 添加任务列表的交互功能
            const setupTaskListInteraction = () => {
              // 根据用户反馈，使用已知有效的选择器
              const checkboxes = previewElement.value.querySelectorAll('.vditor-task input[type="checkbox"]');

              // 处理所有找到的复选框
              checkboxes.forEach((checkbox) => {
                // 强制确保可交互性
                checkbox.disabled = false;
                checkbox.style.pointerEvents = "auto";
                checkbox.style.cursor = "pointer";

                // 找到父级li元素
                const parentLi = checkbox.closest("li");
                if (parentLi) {
                  // 如果已经勾选，添加样式属性
                  if (checkbox.checked) {
                    parentLi.setAttribute("data-task-checked", "true");
                  }

                  // 添加change事件处理
                  checkbox.addEventListener("change", (e) => {
                    const isChecked = e.target.checked;
                    // 更新父元素的数据属性，启用样式
                    parentLi.setAttribute("data-task-checked", isChecked.toString());
                  });
                }
              });
            };

            // 运行初始设置
            setupTaskListInteraction();

            // 设置DOM观察器，处理动态变化
            const observer = new MutationObserver(() => {
              // 延迟短暂时间后运行，确保DOM更新完成
              setTimeout(setupTaskListInteraction, 50);
            });

            // 监听DOM变化
            observer.observe(previewElement.value, {
              childList: true,
              subtree: true,
            });

            // 自定义图片点击放大功能
            const images = previewElement.value.querySelectorAll("img");
            images.forEach((img) => {
              // 避免重复添加事件
              if (!img.getAttribute("data-preview-bound")) {
                img.setAttribute("data-preview-bound", "true");
                img.style.cursor = "pointer"; // 显示为可点击的样式

                img.addEventListener("click", (e) => {
                  e.preventDefault();

                  // 创建图片预览容器
                  const previewContainer = document.createElement("div");
                  previewContainer.className = "image-preview-container";
                  previewContainer.style.position = "fixed";
                  previewContainer.style.top = "0";
                  previewContainer.style.left = "0";
                  previewContainer.style.width = "100%";
                  previewContainer.style.height = "100%";
                  previewContainer.style.backgroundColor = "rgba(0, 0, 0, 0.8)";
                  previewContainer.style.zIndex = "10000";
                  previewContainer.style.display = "flex";
                  previewContainer.style.justifyContent = "center";
                  previewContainer.style.alignItems = "center";
                  previewContainer.style.cursor = "zoom-out";

                  // 创建图片元素
                  const previewImg = document.createElement("img");
                  previewImg.src = img.src;
                  previewImg.style.maxWidth = "90%";
                  previewImg.style.maxHeight = "90%";
                  previewImg.style.objectFit = "contain";
                  previewImg.style.boxShadow = "0 4px 12px rgba(0, 0, 0, 0.5)";
                  previewImg.style.transition = "transform 0.3s ease";

                  // 添加点击关闭功能
                  previewContainer.addEventListener("click", () => {
                    document.body.removeChild(previewContainer);
                  });

                  // 防止点击图片时冒泡事件关闭预览
                  previewImg.addEventListener("click", (e) => {
                    e.stopPropagation();
                  });

                  // 添加到DOM
                  previewContainer.appendChild(previewImg);
                  document.body.appendChild(previewContainer);
                });
              }
            });

            // 标记内容已渲染，并触发渲染完成事件
            contentRendered.value = true;
            emit("rendered");
          },
        });
      }, 10); // 短延迟，确保DOM更新
    } catch (e) {
      console.error("渲染 Markdown 内容时发生错误:", e);
      // 回退到基本的文本显示
      if (previewElement.value) {
        // 内容转义，避免XSS风险
        const safeContent = content.replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;").replace(/"/g, "&quot;").replace(/'/g, "&#039;");
        previewElement.value.innerHTML = `<pre class="whitespace-pre-wrap">${safeContent}</pre>`;
        // 即使是回退渲染，也标记为已渲染
        contentRendered.value = true;
        emit("rendered");
      }
    }
  } else {
    debugLog(props.enableDebug, props.isDev, "renderContentInternal: 预览元素不存在");
    // 在DOM可用时重试（仅重试一次，避免无限循环）
    setTimeout(() => {
      if (previewElement.value && !contentRendered.value) {
        debugLog(props.enableDebug, props.isDev, "延迟后重试渲染");
        renderContentInternal(content);
      }
    }, 150);
  }
};

// 检查内容是否已渲染，提供多种判断方法确保准确性
const isContentRendered = () => {
  // 优先使用明确的渲染状态标记
  if (contentRendered.value) {
    return true;
  }

  // 以下是备用检查逻辑
  if (!previewElement.value) return false;

  // 检查预览元素是否有内容
  const innerHTML = previewElement.value.innerHTML || "";
  // 更可靠的渲染检测 - 检查是否包含vditor特有的元素或类
  const hasVditorContent = innerHTML.includes('class="vditor') || previewElement.value.querySelectorAll(".vditor-reset").length > 0;

  // 如果有内容，也标记为已渲染
  if (hasVditorContent || innerHTML.trim().length > 50) {
    contentRendered.value = true;
    return true;
  }

  return false;
};

// 对外暴露方法，允许父组件调用
defineExpose({
  renderContent,
  isContentRendered,
});

// 组件挂载时尝试渲染内容
onMounted(() => {
  if (props.content) {
    renderContent(props.content);
  }
});
</script>

<template>
  <div class="paste-view-preview">
    <!-- 内容渲染中的加载动画 -->
    <div v-if="props.content && !contentRendered" class="py-10 flex justify-center items-center">
      <svg class="animate-spin h-8 w-8" :class="props.darkMode ? 'text-blue-400' : 'text-primary-500'" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
        <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="3"></circle>
        <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
      </svg>
      <span class="ml-3 text-sm" :class="props.darkMode ? 'text-gray-300' : 'text-gray-500'">正在渲染内容...</span>
    </div>

    <!-- Vditor 预览容器 - 这里是实际的Markdown渲染区域 -->
    <div ref="previewElement" class="vditor-reset markdown-body min-h-[300px]" :class="{ 'opacity-0': !contentRendered }"></div>

    <!-- 后备内容显示 - 当预览元素为空或渲染失败时显示原始Markdown -->
    <div
        v-if="props.content && !isContentRendered() && previewElement && contentRendered === false"
        class="mt-4 p-3 border rounded"
        :class="props.darkMode ? 'border-yellow-600 bg-yellow-900/20' : 'border-yellow-500 bg-yellow-50'"
    >
      <p class="text-sm mb-2" :class="props.darkMode ? 'text-yellow-300' : 'text-yellow-700'">Markdown 渲染失败，显示原始内容：</p>
      <pre class="whitespace-pre-wrap overflow-auto max-h-[600px] p-3 rounded" :class="props.darkMode ? 'text-gray-200 bg-gray-800' : 'text-gray-800 bg-gray-100'">{{
          props.content
        }}</pre>
    </div>

    <!-- 无内容提示 -->
    <p v-if="!props.content" :class="props.darkMode ? 'text-gray-400' : 'text-gray-500'">无内容</p>
  </div>
</template>

<style scoped>
/* Vditor预览样式定制 */
/* 这些样式会被应用到Vditor渲染的内容上，使其更美观并适应暗色/亮色模式 */
:deep(.vditor-reset) {
  font-size: 16px;
  line-height: 1.7;
  padding: 0.5rem;
  transition: all 0.3s ease; /* 增加所有属性的过渡效果 */
  color: v-bind('props.darkMode ? "#d4d4d4" : "#374151"'); /* 显式设置文本颜色 */
  background-color: v-bind('props.darkMode ? "transparent" : "transparent"'); /* 确保背景透明 */
}

/* 确保暗色模式下的特定样式 */
:deep(.vditor-reset--dark) {
  color: #d4d4d4;
  background-color: transparent;
}

/* 确保亮色模式下的特定样式 */
:deep(.vditor-reset--light) {
  color: #374151;
  background-color: transparent;
}

/* 标题样式 */
:deep(.vditor-reset h1, .vditor-reset h2) {
  border-bottom: 1px solid v-bind('props.darkMode ? "#30363d" : "#e5e7eb"');
  padding-bottom: 0.3em;
  margin-top: 1.8em;
  margin-bottom: 1em;
}

:deep(.vditor-reset h1) {
  font-size: 2em;
}

:deep(.vditor-reset h2) {
  font-size: 1.6em;
}

:deep(.vditor-reset h3) {
  font-size: 1.4em;
  margin-top: 1.5em;
  margin-bottom: 0.75em;
}

/* 段落样式 */
:deep(.vditor-reset p) {
  margin-top: 0.75em;
  margin-bottom: 0.75em;
}

/* 行内代码样式 */
:deep(.vditor-reset code:not(.hljs)) {
  background-color: v-bind('props.darkMode ? "#252526" : "#f3f4f6"');
  color: v-bind('props.darkMode ? "#ce9178" : "#ef4444"');
  padding: 0.2em 0.4em;
  border-radius: 3px;
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace;
}

/* 引用块样式 */
:deep(.vditor-reset blockquote) {
  border-left: 4px solid v-bind('props.darkMode ? "#4b5563" : "#e5e7eb"');
  padding: 0.5em 1em;
  margin-left: 0;
  margin-right: 0;
  margin-top: 1em;
  margin-bottom: 1em;
  color: v-bind('props.darkMode ? "#9ca3af" : "#6b7280"');
  background-color: v-bind('props.darkMode ? "#1a1a1a" : "#f9fafb"');
  border-radius: 0.25rem;
}

/* 链接样式 */
:deep(.vditor-reset a) {
  color: v-bind('props.darkMode ? "#3b82f6" : "#2563eb"');
  text-decoration: none;
}

:deep(.vditor-reset a:hover) {
  text-decoration: underline;
}

/* 表格样式 */
:deep(.vditor-reset table) {
  border-collapse: collapse;
  margin: 1.25em 0;
  width: 100%;
}

/* 添加表格容器，确保在小屏幕上可以滚动 */
:deep(.vditor-reset div:has(> table)) {
  overflow-x: auto;
  max-width: 100%;
}

:deep(.vditor-reset table th, .vditor-reset table td) {
  border: 1px solid v-bind('props.darkMode ? "#30363d" : "#e5e7eb"');
  padding: 0.6em 1em;
}

:deep(.vditor-reset table th) {
  background-color: v-bind('props.darkMode ? "#252526" : "#f3f4f6"');
  font-weight: 600;
  color: v-bind('props.darkMode ? "#e2e8f0" : "#374151"');
}

:deep(.vditor-reset table td) {
  background-color: v-bind('props.darkMode ? "#1e1e1e" : "#ffffff"');
}

:deep(.vditor-reset table tr:nth-child(even) td) {
  background-color: v-bind('props.darkMode ? "#252526" : "#f9fafb"');
}

:deep(.vditor-reset table tr:hover td) {
  background-color: v-bind('props.darkMode ? "#2c2c2d" : "#f3f4f6"');
}

/* 代码块样式 */
:deep(.vditor-reset pre) {
  margin: 1.25em 0;
  padding: 1.25em;
  border-radius: 0.375rem;
  background-color: v-bind('props.darkMode ? "#1e1e1e" : "#f8fafc"');
  overflow-x: auto;
  line-height: 1.5;
}

/* 列表样式 */
:deep(.vditor-reset ul, .vditor-reset ol) {
  padding-left: 2em;
  margin: 1em 0;
}

/* 多级列表样式 */
/* 有序列表样式 */
:deep(.vditor-reset ol) {
  list-style-type: decimal;
}

:deep(.vditor-reset ol ol) {
  list-style-type: decimal;
}

:deep(.vditor-reset ol ol ol) {
  list-style-type: decimal;
}

/* 无序列表样式 */
:deep(.vditor-reset ul) {
  list-style-type: disc;
}

:deep(.vditor-reset ul ul) {
  list-style-type: circle;
}

:deep(.vditor-reset ul ul ul) {
  list-style-type: square;
}

/* 图片样式 */
:deep(.vditor-reset img) {
  max-width: 100%;
  margin: 1.25em 0;
  border-radius: 0.25rem;
}

/* 图表样式固定 - 不受暗色模式影响 */
:deep(.diagram-fixed-theme) {
  background-color: white !important; /* 强制使用白色背景 */
  color: #333 !important; /* 强制使用深色文本 */
  filter: none !important; /* 移除任何可能的过滤器效果 */
}

/* 确保mermaid图表使用固定样式 */
:deep(.language-mermaid) {
  background-color: white !important;
  padding: 10px !important;
  border-radius: 4px !important;
  border: 1px solid #e5e7eb !important;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05) !important;
}

/* 流程图样式固定 */
:deep(.language-flow) {
  background-color: white !important;
  padding: 10px !important;
  border-radius: 4px !important;
  border: 1px solid #e5e7eb !important;
}

/* PlantUML图表样式固定 */
:deep(.language-plantuml) {
  background-color: white !important;
  padding: 10px !important;
  border-radius: 4px !important;
  border: 1px solid #e5e7eb !important;
}

/* 甘特图样式固定 */
:deep(.language-gantt) {
  background-color: white !important;
  padding: 10px !important;
  border-radius: 4px !important;
  border: 1px solid #e5e7eb !important;
}

/* 图表内的SVG元素样式固定 */
:deep(.diagram-fixed-theme svg) {
  background-color: white !important;
}

/* 针对手机屏幕的响应式调整 */
@media (max-width: 640px) {
  :deep(.vditor-reset) {
    font-size: 15px;
    padding: 0.25rem;
  }

  :deep(.vditor-reset pre) {
    padding: 0.75em;
  }

  :deep(.vditor-reset table) {
    font-size: 14px;
  }
}

/* 中等屏幕 */
@media (min-width: 641px) and (max-width: 1024px) {
  :deep(.vditor-reset) {
    padding: 0.75rem;
  }
}

/* 大屏幕额外优化 */
@media (min-width: 1280px) {
  :deep(.vditor-reset) {
    font-size: 17px;
    padding: 1rem;
  }

  :deep(.vditor-reset pre) {
    margin: 1.5em 0;
    padding: 1.5em;
  }
}

/* 任务列表样式增强 */
:deep(.vditor-reset ul li input[type="checkbox"]) {
  cursor: pointer !important;
  pointer-events: auto !important;
  margin-right: 0.5em;
  width: 1.2em;
  height: 1.2em;
  vertical-align: middle;
  position: relative;
  top: -0.1em;
}

:deep(.vditor-reset ul li) {
  list-style-type: none;
  position: relative;
}

/* 确保任务列表项交互状态更明显 */
:deep(.vditor-reset ul li[data-task-checked="true"]) {
  text-decoration: line-through;
  color: v-bind('props.darkMode ? "#6b7280" : "#9ca3af"');
}

:deep(.vditor-reset ul li input[type="checkbox"]:hover),
:deep(.vditor-reset ul li input[type="checkbox"]:focus) {
  outline: 1px solid v-bind('props.darkMode ? "#3b82f6" : "#2563eb"');
  box-shadow: 0 0 0 2px rgba(59, 130, 246, 0.3);
}

/* 确保任务列表在暗色模式下有正确的颜色 */
:deep(.vditor-reset--dark ul li input[type="checkbox"]) {
  background-color: #1e1e1e;
  border-color: #4b5563;
}

:deep(.vditor-reset--light ul li input[type="checkbox"]) {
  background-color: #ffffff;
  border-color: #d1d5db;
}
</style>
