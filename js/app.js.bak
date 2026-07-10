// ============================================
// 药品信息查询 App - Main Application
// ============================================

const { createApp, ref, computed, watch, nextTick, onMounted } = Vue;

// --- Data Loading ---
const drugData = ref([]);
let fuseInstance = null;
let dataLoaded = false;

async function loadDrugData() {
  try {
    let data;
    if (typeof INLINE_DRUG_DATA !== "undefined" && INLINE_DRUG_DATA) {
      data = INLINE_DRUG_DATA;
    } else {
      const base = APP_CONFIG.basePath || "";
      const resp = await fetch(base + "data/drugs.json");
      data = await resp.json();
    }
    drugData.value = data;
    dataLoaded = true;

    fuseInstance = new Fuse(data, {
      keys: [
        { name: "name", weight: 10 },
        { name: "genericName", weight: 8 },
        { name: "enName", weight: 5 },
        { name: "summary", weight: 6 },
        { name: "manufacturer", weight: 3 },
        { name: "category", weight: 4 },
        { name: "sections.prescribing_info.content.indications.body", weight: 5 },
        { name: "sections.prescribing_info.content.dosage.body", weight: 3 },
        { name: "sections.prescribing_info.content.adverse_reactions.body", weight: 3 },
        { name: "sections.prescribing_info.content.contraindications.body", weight: 4 },
        { name: "sections.faq.content.*.question", weight: 5 },
        { name: "sections.faq.content.*.answer", weight: 4 }
      ],
      threshold: 0.4,
      includeScore: true,
      minMatchCharLength: 1
    });
  } catch (e) {
    console.error("Failed to load drug data:", e);
  }
}

// --- Categories ---
function getCategories() {
  const cats = {};
  const categoryOrder = ["维生素D₂", "鲑降钙素", "班布特罗", "学术文献"];
  drugData.value.forEach(d => {
    if (!cats[d.category]) cats[d.category] = { count: 0, icon: getCategoryIcon(d.category) };
    cats[d.category].count++;
  });
  const result = categoryOrder
    .filter(name => cats[name])
    .map(name => ({ name, count: cats[name].count, icon: cats[name].icon }));
  result.unshift({ name: "全部", count: drugData.value.length, icon: "📁" });
  return result;
}

function getCategoryIcon(category) {
  const iconMap = {
    "抗血小板药": "🞧8",
    "心血管系统": "❤️",
    "质子泵抑制剂": "🔬",
    "消化系统": "🏥",
    "口服降糖药": "💉",
    "内分泌系统": "⚖️",
    "鲑降钙素": "🐟",
    "学术文献": "📖",
    "班布特罗": "💊"
  };
  for (const [key, icon] of Object.entries(iconMap)) {
    if (category.includes(key)) return icon;
  }
  return "💊";
}

// --- App Component ---
const App = {
  setup() {
    const hasScroll = ref(false);
    const showSearch = ref(false);
    const searchQuery = ref("");
    const searchResults = ref([]);
    const searchInput = ref(null);
    const currentRoute = ref("home");
    const activeCategory = ref("全部");
    const activeSection = ref("");
    const currentDrugId = ref("");
    const toast = ref({ show: false, message: "", timer: null });

    // Drug group selection
    const showDrugSelector = ref(false);
    const selectedDrugName = ref("");

    function showToast(msg, duration) {
      duration = duration || 2500;
      if (toast.value.timer) clearTimeout(toast.value.timer);
      toast.value.message = msg;
      toast.value.show = true;
      toast.value.timer = setTimeout(() => { toast.value.show = false; }, duration);
    }

    // Chat state
    const chatMessages = ref([]);
    const chatInput = ref("");
    const isThinking = ref(false);
    const apiKeyInput = ref("");
    const apiKeySet = ref(!!localStorage.getItem("deepseek_api_key"));
    const isRecording = ref(false);

    const pageTitle = computed(() => {
      if (currentRoute.value === "home") return "海鲸药品信息查询";
      if (currentRoute.value === "detail" && currentDrug.value) return currentDrug.value.name;
      if (currentRoute.value === "section-menu" && currentDrug.value) return currentDrug.value.name + " - 资料选择";
      if (currentRoute.value === "chat") return "AI 药品助手";
      return "海鲸药品信息查询";
    });

    const filteredDrugs = computed(() => {
      if (activeCategory.value === "全部") return drugData.value;
      return drugData.value.filter(d => d.category === activeCategory.value);
    });

    // displayDrugs: merge vitamin-d2 and vitamin-d2-400 into one group entry
    const displayDrugs = computed(() => {
      const list = filteredDrugs.value;
      const result = [];
      let groupAdded = false;
      for (const drug of list) {
        if (drug.id === "vitamin-d2" || drug.id === "vitamin-d2-400") {
          if (!groupAdded) {
            result.push({
              id: "vitamin-d2-group",
              name: "维生素D₂软胶囊",
              genericName: "维生素D₂",
              enName: "Vitamin D₂ Soft Capsules",
              category: drug.category,
              category2: drug.category2,
              manufacturer: drug.manufacturer,
              specification: "0.01mg（400单位） / 0.125mg（5000单位）",
              approvalNumber: "国药准字H20247143 / 国药准字H32023837",
              drugCode: "",
              summary: "维生素D₂用于维生素D缺乏的预防与治疗。南京海鲸药业生产。点击查看400单位和5000单位两种规格。",
              fileUrl: "",
              fullText: "",
              sections: drug.sections,
              isGroup: true
            });
            groupAdded = true;
          }
        } else {
          result.push(drug);
        }
      }
      return result;
    });

    const drugItems = computed(() => {
      return displayDrugs.value.filter(d => d.category !== "学术文献");
    });

    const academicItems = computed(() => {
      return displayDrugs.value.filter(d => d.category === "学术文献");
    });

    const currentDrug = computed(() => {
      return drugData.value.find(d => d.id === currentDrugId.value) || null;
    });

    const currentSectionObj = computed(() => {
      if (!currentDrug.value || !activeSection.value) return null;
      return currentDrug.value.sections[activeSection.value] || null;
    });

    const categories = computed(() => getCategories());

    function toggleChat() {
      if (currentRoute.value === "chat") {
        currentRoute.value = "home";
        return;
      }
      currentRoute.value = "chat";
      if (chatMessages.value.length === 0) {
        chatMessages.value.push({
          role: "assistant",
          text: "您好，欢迎来到海鲸药业学术支持查询小工具。我是AI学习助手，回答范围限定于公司内部上传的资料库（产品资料、临床数据、学术文献等），确保信息的准确与合规。请问有什么可以帮您？",
          time: new Date().toLocaleTimeString([], { hour: "2-digit", minute: "2-digit" })
        });
      }
      scrollToTop();
    }

    async function sendMessage() {
      const text = chatInput.value.trim();
      if (!text) return;
      chatInput.value = "";
      chatMessages.value.push({ role: "user", text, time: new Date().toLocaleTimeString([], { hour: "2-digit", minute: "2-digit" }) });
      isThinking.value = true;
      try {
        let key = localStorage.getItem("deepseek_api_key");
        if (!key) throw new Error("请先设置 API Key");
        const context = drugData.value.map(d => d.fullText || "").filter(Boolean).join("\n\n---\n\n");
        // Build conversation history (last 6 exchanges to save tokens)
        const historyMsgs = [];
        const msgs = chatMessages.value;
        const startIdx = Math.max(0, msgs.length - 12); // last 6 exchanges = 12 messages
        for (let i = startIdx; i < msgs.length; i++) {
          const m = msgs[i];
          if (m.role === "user" || m.role === "assistant") {
            historyMsgs.push({ role: m.role, content: m.text });
          }
        }
        const resp = await fetch("https://api.deepseek.com/v1/chat/completions", {
          method: "POST",
          headers: { "Content-Type": "application/json", "Authorization": "Bearer " + key },
          body: JSON.stringify({
            model: "deepseek-chat",
            messages: [
              { role: "system", content: "你是海鲸药业的AI学术小助手。请基于以下药品资料库的内容回答医药代表的问题。如果资料库中没有相关信息，请如实告知。不要编造信息。\n\n资料库内容：\n" + context },
              ...historyMsgs,
              { role: "user", content: text }
            ],
            temperature: 0.3,
            max_tokens: 2048,
            stream: false
          })
        });
        if (!resp.ok) throw new Error("API 错误: " + resp.status);
        const data = await resp.json();
        const reply = data.choices?.[0]?.message?.content || "抱歉，暂时无法获取回答。";
        chatMessages.value.push({ role: "assistant", text: reply, time: new Date().toLocaleTimeString([], { hour: "2-digit", minute: "2-digit" }) });
      } catch (e) {
        chatMessages.value.push({ role: "assistant", text: "错误: " + e.message, time: new Date().toLocaleTimeString([], { hour: "2-digit", minute: "2-digit" }) });
      }
      isThinking.value = false;
      await nextTick();
    }

    function saveApiKey() {
      const key = apiKeyInput.value.trim();
      if (key) {
        localStorage.setItem("deepseek_api_key", key);
        apiKeySet.value = true;
        showToast("API Key 已保存");
      }
    }

    function changeApiKey() {
      apiKeySet.value = false;
      apiKeyInput.value = "";
    }

    function startVoice() {
      showToast("语音输入功能正在开发中");
    }

    function filterByCategory(cat) {
      activeCategory.value = cat;
      currentDrugId.value = "";
      activeSection.value = "";
    }

    function viewDrug(id) {
      if (id === "vitamin-d2-group") {
        showDrugSelector.value = true;
        selectedDrugName.value = "维生素D₂软胶囊";
        return;
      }
      currentDrugId.value = id;
      const drug = drugData.value.find(d => d.id === id);
      if (drug && drug.category === "学术文献") {
        const keys = Object.keys(drug.sections || {});
        activeSection.value = keys[0] || "";
        currentRoute.value = "detail";
      } else {
        currentRoute.value = "section-menu";
      }
      scrollToTop();
    }

    function selectDrugSub(id) {
      showDrugSelector.value = false;
      selectedDrugName.value = "";
      currentDrugId.value = id;
      currentRoute.value = "section-menu";
      scrollToTop();
    }

    function goToSection(key) {
      const drug = currentDrug.value;
      if (!drug || !drug.sections || !drug.sections[key]) {
        showToast("栏目内容待添加", 2000);
        return;
      }
      const section = drug.sections[key];
      const contentKeys = Object.keys(section.content || {});
      if (contentKeys.length === 0) {
        showToast("栏目内容待添加", 2000);
        return;
      }
      activeSection.value = key;
      currentRoute.value = "detail";
      scrollToTop();
    }

        function goBack() {
      if (currentRoute.value === "chat") {
        currentRoute.value = "home";
        return;
      }
      if (currentRoute.value === "detail") {
        const drug = currentDrug.value;
        if (drug && drug.category === "学术文献") {
          currentRoute.value = "home";
          currentDrugId.value = "";
        } else {
          currentRoute.value = "section-menu";
        }
        activeSection.value = "";
        window.scrollTo({ top: 0, behavior: "instant" });
        return;
      }
      if (currentRoute.value === "section-menu") {
        currentRoute.value = "home";
        currentDrugId.value = "";
        window.scrollTo({ top: 0, behavior: "instant" });
        return;
      }
    }

    
            function getFileUrl(url) {
      const base = APP_CONFIG.basePath || '';
      return base + url;
    }

    function openFile(url) {
      if (!url) {
        showToast("文件链接无效", 2000);
        return;
      }
      const fullUrl = getFileUrl(url);
      window.location.href = fullUrl;
    }
function scrollToTop() {
      window.scrollTo({ top: 0, behavior: "instant" });
    }

    function renderMarkdown(text) {
      if (!text) return "";
      text = text.replace(/\\n/g, "\n");
      let html = text.replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;").replace(/\*\*(.+?)\*\*/g, "<strong>$1</strong>");
      const lines = html.split("\n");
      for (let li = 0; li < lines.length; li++) {
        lines[li] = lines[li].replace(/^#+\s+/, "");
      }
      const blocks = [];
      let i = 0;
      while (i < lines.length) {
        const line = lines[i].trim();
        if (!line) { i++; continue; }
        if (line.startsWith("|") && line.endsWith("|") && line.length > 2) {
          const tableRows = [];
          while (i < lines.length) {
            const l = lines[i].trim();
            if (!l.startsWith("|")) break;
            const cells = l.split("|").filter(c => c.trim());
            if (cells.length > 0 && cells.every(c => /^[\s:-]+$/.test(c.trim()))) { i++; continue; }
            tableRows.push(l); i++;
          }
          let tableHtml = "<div style='overflow-x:auto'><table style='width:auto;white-space:nowrap'>";
          if (tableRows.length > 0) {
            const headers = tableRows[0].split("|").filter(c => c.trim());
            if (headers.length > 0) {
              tableHtml += "<thead><tr>" + headers.map(h => "<th style='white-space:nowrap'>" + h.trim() + "</th>").join("") + "</tr></thead>";
            }
            if (tableRows.length > 1) {
              tableHtml += "<tbody>";
              for (let r = 1; r < tableRows.length; r++) {
                const cells = tableRows[r].split("|").filter(c => c.trim());
                tableHtml += "<tr>" + cells.map(c => "<td style='white-space:nowrap'>" + c.trim() + "</td>").join("") + "</tr>";
              }
              tableHtml += "</tbody>";
            }
          }
          tableHtml += "</table></div>";
          blocks.push(tableHtml);
          continue;
        }
        if (/^\d+[\.\u3002]\s/.test(line)) {
          const listItems = [];
          while (i < lines.length) {
            const l = lines[i].trim();
            if (/^\d+[\.\u3002]\s/.test(l)) { listItems.push("<li>" + l.replace(/^\d+[\.\u3002]\s/, "") + "</li>"); i++; }
            else break;
          }
          blocks.push("<ol>" + listItems.join("") + "</ol>");
          continue;
        }
        if (/^[-*\u2022]\s/.test(line)) {
          const listItems = [];
          while (i < lines.length) {
            const l = lines[i].trim();
            if (/^[-*\u2022]\s/.test(l)) { listItems.push("<li>" + l.replace(/^[-*\u2022]\s/, "") + "</li>"); i++; }
            else break;
          }
          blocks.push("<ul>" + listItems.join("") + "</ul>");
          continue;
        }
        const paraLines = [];
        while (i < lines.length) {
          const l = lines[i].trim();
          if (!l) break;
          if (l.startsWith("|") || /^\d+[\.\u3002]\s/.test(l) || /^[-*\u2022]\s/.test(l)) break;
          paraLines.push(l); i++;
        }
        if (paraLines.length > 0) blocks.push("<p>" + paraLines.join("<br>") + "</p>");
      }
      return blocks.join("\n");
    }

    function onScroll() { hasScroll.value = window.scrollY > 5; }

    function handlePopState() {
      if (currentRoute.value === "detail") {
        currentRoute.value = "home";
        currentDrugId.value = "";
        activeSection.value = "";
      } else if (showSearch.value) {
        showSearch.value = false;
        searchQuery.value = "";
        searchResults.value = [];
      }
    }

    onMounted(() => {
      loadDrugData();
      window.addEventListener("scroll", onScroll, { passive: true });
      window.addEventListener("popstate", handlePopState);
    });

    return {
      hasScroll, showSearch, searchQuery, searchResults, searchInput,
      currentRoute, activeCategory, activeSection, currentDrugId, toast,
      pageTitle, filteredDrugs, currentDrug, currentSectionObj, categories,
      filterByCategory, viewDrug, goBack, goToSection, renderMarkdown, scrollToTop,
      showDrugSelector, selectedDrugName, selectDrugSub, displayDrugs,
      drugItems, academicItems,
      toggleChat, sendMessage, chatMessages, chatInput,
      isThinking, apiKeyInput, apiKeySet, saveApiKey, changeApiKey,
      showToast, isRecording, startVoice, getFileUrl, openFile
    };
  }
};

const app = createApp(App);
app.mount("#app");





