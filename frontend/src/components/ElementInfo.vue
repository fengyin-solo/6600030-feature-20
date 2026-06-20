<script setup lang="ts">
import { computed } from 'vue';
import { useFEAStore } from '../store/fea';

const store = useFEAStore();

const selectedEl = computed(() => {
  if (store.selectedElement === null) return null;
  return store.model.elements.find((e) => e.id === store.selectedElement) || null;
});

const node1 = computed(() => {
  if (!selectedEl.value) return null;
  return store.model.nodes.find((n) => n.id === selectedEl.value!.nodeIds[0]) || null;
});

const node2 = computed(() => {
  if (!selectedEl.value) return null;
  return store.model.nodes.find((n) => n.id === selectedEl.value!.nodeIds[1]) || null;
});

const length = computed(() => {
  if (!node1.value || !node2.value) return 0;
  const dx = node2.value.x - node1.value.x;
  const dy = node2.value.y - node1.value.y;
  return Math.sqrt(dx * dx + dy * dy);
});

const angle = computed(() => {
  if (!node1.value || !node2.value) return 0;
  const dx = node2.value.x - node1.value.x;
  const dy = node2.value.y - node1.value.y;
  return (Math.atan2(dy, dx) * 180) / Math.PI;
});

const color = computed(() => {
  if (store.selectedElement === null) return '#6b7280';
  return store.elementColors.get(store.selectedElement) || '#6b7280';
});

// ─── 基础参数与常量 ─────────────────────────────────────────────────────────
const DEFAULT_YIELD = 235e6;
const ADULT_WEIGHT_N = 650;
const CAR_WEIGHT_N = 15000;
const HAIR_DIAMETER_MM = 0.07;
const PAPER_THICKNESS_MM = 0.1;

const hasResult = computed(() => store.result !== null);

const yieldStrength = computed(() => {
  if (!selectedEl.value) return DEFAULT_YIELD;
  return selectedEl.value.yieldStrength ?? DEFAULT_YIELD;
});

// ─── 1. 受力状态 ────────────────────────────────────────────────────────────
const forceState = computed<{ label: string; color: string; desc: string; icon: string }>(() => {
  if (!selectedEl.value) return { label: '—', color: '#6b7280', desc: '', icon: '❓' };
  const f = selectedEl.value.force;
  if (Math.abs(f) < 1) {
    return {
      label: '近乎不受力',
      color: '#6b7280',
      desc: '该杆件几乎不承受任何载荷，属于冗余杆件，受力可以忽略不计。',
      icon: '⚪',
    };
  }
  if (f > 0) {
    return {
      label: '受拉',
      color: '#38bdf8',
      desc: '该杆件正被两端向外拉扯，就像拔河用的绳子一样被拉长。拉力作用下，杆件一般不会突然失稳，较为安全。',
      icon: '🔗',
    };
  }
  return {
    label: '受压',
    color: '#f59e0b',
    desc: '该杆件正被两端向内挤压，就像支撑重物的柱子一样被压缩。⚠️ 受压杆件有「失稳弯曲」的风险，细长杆件尤其需要注意！',
    icon: '🪵',
  };
});

// ─── 2. 力的生活类比 ────────────────────────────────────────────────────────
const forceAnalogy = computed<{ text: string; icon: string }>(() => {
  if (!selectedEl.value) return { text: '—', icon: '' };
  const fAbs = Math.abs(selectedEl.value.force);
  if (fAbs < 1) return { text: '几乎为零，相当于一枚硬币的重量', icon: '🪙' };
  if (fAbs < 100) return { text: `约相当于 ${(fAbs / 10).toFixed(0)} kg 重物的压力`, icon: '📦' };
  if (fAbs < 1000) return { text: `约相当于 ${(fAbs / ADULT_WEIGHT_N).toFixed(1)} 个成年人的重量`, icon: '👤' };
  if (fAbs < 10000) return { text: `约相当于 ${(fAbs / ADULT_WEIGHT_N).toFixed(0)} 个成年人站在上面`, icon: '👥' };
  if (fAbs < 100000) return { text: `约相当于 ${(fAbs / CAR_WEIGHT_N).toFixed(1)} 辆小汽车的重量`, icon: '🚗' };
  return { text: `约相当于 ${(fAbs / CAR_WEIGHT_N).toFixed(0)} 辆小汽车叠在一起`, icon: '🚛' };
});

// ─── 3. 应力水平（相对最大应力） ───────────────────────────────────────────
const stressRatioToMax = computed(() => {
  if (!selectedEl.value || store.maxStress <= 0) return 0;
  return Math.abs(selectedEl.value.stress) / store.maxStress;
});

const stressRank = computed<{ level: string; desc: string; color: string }>(() => {
  const r = stressRatioToMax.value;
  if (r <= 0) return { level: '无数据', desc: '当前无有效应力数据。', color: '#6b7280' };
  if (r >= 0.999) return {
    level: '🔴 关键部位',
    desc: '这是整个结构中应力最大的杆件，相当于「顶梁柱」，一旦出问题整个结构会受严重影响！',
    color: '#ef4444',
  };
  if (r >= 0.75) return {
    level: '🟠 高应力区',
    desc: '该杆件应力很高，接近结构最大值，属于主要承重杆件，建议重点关注。',
    color: '#f97316',
  };
  if (r >= 0.4) return {
    level: '🟡 中等应力',
    desc: '该杆件应力处于中等水平，正常工作的一员，不是最薄弱的环节。',
    color: '#eab308',
  };
  return {
    level: '🟢 低应力',
    desc: '该杆件应力较低，储备充足，不是结构中的主要受力部位。',
    color: '#22c55e',
  };
});

// ─── 4. 变形解读与类比 ──────────────────────────────────────────────────────
const strainPerMille = computed(() => {
  if (!selectedEl.value) return 0;
  return Math.abs(selectedEl.value.strain) * 1000;
});

const absoluteDeformationMM = computed(() => {
  if (!selectedEl.value) return 0;
  return Math.abs(selectedEl.value.strain) * length.value * 1000;
});

const deformationAnalogy = computed<{ text: string; level: string }>(() => {
  const mm = absoluteDeformationMM.value;
  if (mm < 0.01) return { text: '变形极小，肉眼完全无法察觉', level: '极小' };
  if (mm < HAIR_DIAMETER_MM / 2) return { text: `变形约为头发丝直径的 ${(mm / HAIR_DIAMETER_MM * 100).toFixed(0)}%`, level: '微小' };
  if (mm < HAIR_DIAMETER_MM) return { text: `不到一根头发丝（${HAIR_DIAMETER_MM}mm）的粗细`, level: '细小' };
  if (mm < PAPER_THICKNESS_MM * 3) return { text: `约 ${(mm / HAIR_DIAMETER_MM).toFixed(1)} 根头发丝的厚度`, level: '轻微' };
  if (mm < 1) return { text: `约 ${(mm / PAPER_THICKNESS_MM).toFixed(1)} 张 A4 纸叠起来的厚度`, level: '明显' };
  return { text: `变形已达 ${mm.toFixed(2)}mm，肉眼可见！`, level: '较大' };
});

// ─── 5. 杆件类型判断 ────────────────────────────────────────────────────────
const elementRole = computed<{ type: string; desc: string; icon: string }>(() => {
  if (!node1.value || !node2.value) return { type: '—', desc: '', icon: '' };
  const absAngle = Math.abs(angle.value);
  const f = selectedEl.value?.force ?? 0;

  if (absAngle < 10 || absAngle > 170) {
    if (f > 0) return {
      type: '水平拉杆',
      desc: '这类杆件像「晾衣绳」一样水平拉着，抵抗水平方向的拉力。',
      icon: '↔️',
    };
    return {
      type: '水平压杆',
      desc: '这类杆件水平方向受压，类似横梁的作用，需注意弯曲失稳。',
      icon: '⟷',
    };
  }
  if (absAngle > 80 && absAngle < 100) {
    if (f < 0) return {
      type: '竖向承重柱',
      desc: '这类杆件就像「立柱子」一样竖向支撑，把上面的重量传到基础上。',
      icon: '⬆️',
    };
    return {
      type: '竖向拉杆',
      desc: '这类杆件竖向受拉，像链条一样把上面的结构吊住。',
      icon: '⬇️',
    };
  }
  if (f < 0) return {
    type: '斜向支撑（压）',
    desc: '斜向受压的支撑杆件，类似建筑的「斜撑」，增强结构稳定性。',
    icon: '↗️',
  };
  return {
    type: '斜向拉杆',
    desc: '斜向受拉的杆件，像钢索一样提供斜向拉力，抵抗变形。',
    icon: '↘️',
  };
});

// ─── 6. 应力比 / 安全系数 / 风险等级 ───────────────────────────────────────
const utilization = computed(() => {
  if (!selectedEl.value || yieldStrength.value <= 0) return 0;
  return Math.abs(selectedEl.value.stress) / yieldStrength.value;
});

const safetyFactor = computed(() => {
  const u = utilization.value;
  if (u <= 0) return Infinity;
  return 1 / u;
});

const utilizationPct = computed(() => Math.min(utilization.value * 100, 100));

interface RiskInfo {
  level: string;
  color: string;
  bg: string;
  desc: string;
  emoji: string;
  advice: string[];
}

const risk = computed<RiskInfo>(() => {
  const u = utilization.value;
  const pct = (u * 100).toFixed(1);
  if (u < 0.3) {
    return {
      level: '非常安全',
      color: '#16a34a',
      bg: 'rgba(22,163,74,0.12)',
      emoji: '✅',
      desc: `材料只用了 ${pct}% 的承载力，就像一个能扛 100kg 的人只扛了不到 30kg，非常轻松。`,
      advice: ['无需特别关注', '当前设计很保守', '可考虑适当减小截面以节约材料'],
    };
  }
  if (u < 0.5) {
    return {
      level: '安全',
      color: '#22c55e',
      bg: 'rgba(34,197,94,0.12)',
      emoji: '🟢',
      desc: `材料用了约 ${pct}% 的承载力，相当于能扛 100kg 的人扛了 30-50kg，很稳。`,
      advice: ['正常工作状态', '建议按常规周期检查', '安全储备充足'],
    };
  }
  if (u < 0.7) {
    return {
      level: '正常使用',
      color: '#84cc16',
      bg: 'rgba(132,204,22,0.12)',
      emoji: '🟡',
      desc: `材料用了 ${pct}% 的承载力，相当于一个人扛了 50-70kg，还在合理范围内。`,
      advice: ['处于正常工作区间', '建议定期巡检', '避免超载使用'],
    };
  }
  if (u < 0.85) {
    return {
      level: '警告',
      color: '#eab308',
      bg: 'rgba(234,179,8,0.14)',
      emoji: '⚠️',
      desc: `材料已用了 ${pct}% 的承载力！就像一个人扛了 70-85kg，开始吃力了。`,
      advice: ['应力水平偏高，需密切关注', '避免超载或冲击载荷', '建议进行加固设计评估'],
    };
  }
  if (u < 1.0) {
    return {
      level: '高风险',
      color: '#f97316',
      bg: 'rgba(249,115,22,0.16)',
      emoji: '🔥',
      desc: `材料已用了 ${pct}% 的承载力！接近极限，就像一个人快扛不动了。`,
      advice: ['🚨 应力接近材料极限，存在失效风险', '强烈建议降低载荷或加固杆件', '受压杆件需特别检查失稳可能性', '应尽快进行结构安全评估'],
    };
  }
  return {
    level: '危险！',
    color: '#ef4444',
    bg: 'rgba(239,68,68,0.18)',
    emoji: '💥',
    desc: `材料已超载 ${((u - 1) * 100).toFixed(1)}%！超过了屈服强度，材料可能已经永久变形甚至断裂！`,
    advice: ['🚨🚨🚨 立即停止使用！', '该杆件可能已发生永久塑性变形', '必须紧急加固或更换杆件', '建议全面检查整个结构的安全性'],
  };
});

// ─── 7. 受压杆件失稳专项评估 ───────────────────────────────────────────────
const slendernessRatio = computed(() => {
  if (!selectedEl.value || length.value <= 0) return 0;
  const radius = Math.sqrt(selectedEl.value.area / Math.PI);
  return length.value / radius;
});

const bucklingRisk = computed<{ level: string; color: string; desc: string } | null>(() => {
  if (!selectedEl.value || selectedEl.value.force >= 0) return null;
  const sr = slendernessRatio.value;
  if (sr < 50) return {
    level: '低',
    color: '#22c55e',
    desc: '杆件较粗短，不容易弯曲失稳，主要风险是材料屈服。',
  };
  if (sr < 120) return {
    level: '中等',
    color: '#eab308',
    desc: '杆件长细比适中，需同时关注材料强度和失稳风险。',
  };
  if (sr < 200) return {
    level: '较高',
    color: '#f97316',
    desc: '杆件较为细长，失稳风险不容忽视！建议增加侧向支撑或加大截面。',
  };
  return {
    level: '极高',
    color: '#ef4444',
    desc: '杆件非常细长，极易发生弯曲失稳！即使应力不高，也可能突然弯折破坏，必须增加支撑。',
  };
});

// ─── 8. 一句话总结 ──────────────────────────────────────────────────────────
const oneLineSummary = computed<string>(() => {
  if (!selectedEl.value || !hasResult.value) return '';
  const f = selectedEl.value.force;
  const u = utilization.value;
  const sr = stressRatioToMax.value;

  const forceDir = f > 0 ? '被拉伸（受拉）' : f < 0 ? '被压缩（受压）' : '不受力';
  const riskWord = u < 0.5 ? '安全' : u < 0.85 ? '需关注' : '有风险';

  if (Math.abs(f) < 1) {
    return '这根杆件几乎不受力，对结构承载力贡献很小。';
  }

  let summary = `这是一根${elementRole.value.type}，正在${forceDir}，目前状态${riskWord}`;
  if (sr >= 0.999) summary += '，是整个结构最关键的受力杆件。';
  else if (sr >= 0.75) summary += '，属于主要承重杆件。';
  else if (sr < 0.2) summary += '，不是主要承重部分。';
  else summary += '。';

  return summary;
});
</script>

<template>
  <div class="bg-slate-800 rounded-lg p-4">
    <h3 class="text-sm font-bold text-slate-200 border-b border-slate-700 pb-2 mb-3">
      单元详情
    </h3>

    <div v-if="!selectedEl" class="text-xs text-slate-500 text-center py-6">
      <div class="text-3xl mb-2">👆</div>
      点击画布中的任意杆件查看详细信息
    </div>

    <div v-else class="space-y-3 text-xs">
      <!-- ─── 基础信息 ───────────────────────────────────────────────────── -->
      <div class="flex items-center gap-2 mb-1">
        <div class="w-4 h-4 rounded" :style="{ backgroundColor: color }" />
        <span class="text-slate-300 font-medium text-sm">单元 #{{ selectedEl.id }}</span>
        <span
          class="ml-auto px-2 py-0.5 rounded-full text-[10px] font-bold"
          :style="{ backgroundColor: forceState.color + '22', color: forceState.color }"
        >
          {{ forceState.icon }} {{ forceState.label }}
        </span>
      </div>

      <div class="grid grid-cols-2 gap-2">
        <div class="bg-slate-900 rounded p-2">
          <div class="text-slate-500 text-[10px]">连接节点</div>
          <div class="text-sm font-mono text-slate-200">
            {{ selectedEl.nodeIds[0] }} → {{ selectedEl.nodeIds[1] }}
          </div>
        </div>
        <div class="bg-slate-900 rounded p-2">
          <div class="text-slate-500 text-[10px]">长度 / 角度</div>
          <div class="text-sm font-mono text-slate-200">
            {{ length.toFixed(3) }}m / {{ angle.toFixed(1) }}°
          </div>
        </div>
        <div class="bg-slate-900 rounded p-2">
          <div class="text-slate-500 text-[10px]">截面积</div>
          <div class="text-sm font-mono text-slate-200">
            {{ (selectedEl.area * 1e6).toFixed(0) }} mm²
          </div>
        </div>
        <div class="bg-slate-900 rounded p-2">
          <div class="text-slate-500 text-[10px]">弹性模量</div>
          <div class="text-sm font-mono text-slate-200">
            {{ (selectedEl.youngsModulus / 1e9).toFixed(0) }} GPa
          </div>
        </div>
      </div>

      <!-- ─── 计算结果 ───────────────────────────────────────────────────── -->
      <div v-if="hasResult" class="border-t border-slate-700 pt-2">
        <div class="text-slate-400 mb-2 flex items-center gap-1">
          <span>📊</span> 计算结果
        </div>
        <div class="grid grid-cols-3 gap-2">
          <div class="bg-slate-900 rounded p-2">
            <div class="text-slate-500 text-[10px]">应力 σ</div>
            <div class="text-sm font-bold" :style="{ color }">
              {{ (selectedEl.stress / 1e6).toFixed(2) }}
              <span class="text-[10px] text-slate-500 font-normal">MPa</span>
            </div>
          </div>
          <div class="bg-slate-900 rounded p-2">
            <div class="text-slate-500 text-[10px]">应变 ε</div>
            <div class="text-sm font-bold text-sky-400">
              {{ (selectedEl.strain * 100).toFixed(4) }}
              <span class="text-[10px] text-slate-500 font-normal">%</span>
            </div>
          </div>
          <div class="bg-slate-900 rounded p-2">
            <div class="text-slate-500 text-[10px]">轴力 N</div>
            <div class="text-sm font-bold text-amber-400">
              {{ (selectedEl.force / 1000).toFixed(2) }}
              <span class="text-[10px] text-slate-500 font-normal">kN</span>
            </div>
          </div>
        </div>
      </div>

      <!-- ─── 一句话总结 ─────────────────────────────────────────────────── -->
      <div
        v-if="hasResult"
        class="rounded-lg p-3 border-l-4"
        :style="{ backgroundColor: risk.bg, borderColor: risk.color }"
      >
        <div class="flex items-start gap-2">
          <span class="text-lg">{{ risk.emoji }}</span>
          <div>
            <div class="font-bold text-[11px] mb-0.5" :style="{ color: risk.color }">
              {{ risk.level }}
            </div>
            <p class="text-[11px] leading-relaxed text-slate-300">
              {{ oneLineSummary }}
            </p>
          </div>
        </div>
      </div>

      <!-- ─── 结果解读（通俗版） ────────────────────────────────────────── -->
      <div v-if="hasResult" class="border-t border-slate-700 pt-2">
        <div class="text-slate-400 mb-2 flex items-center gap-1">
          <span>💡</span> 结果解读
        </div>
        <div class="space-y-2">
          <!-- 杆件角色 -->
          <div class="bg-slate-900 rounded-lg p-2.5">
            <div class="flex items-center gap-2 mb-1">
              <span class="text-base">{{ elementRole.icon }}</span>
              <span class="font-bold text-slate-200 text-[11px]">{{ elementRole.type }}</span>
            </div>
            <p class="text-[11px] leading-relaxed text-slate-400">{{ elementRole.desc }}</p>
          </div>

          <!-- 受力状态 + 类比 -->
          <div class="bg-slate-900 rounded-lg p-2.5">
            <div class="flex items-center justify-between mb-1">
              <span class="font-bold" :style="{ color: forceState.color }">
                {{ forceState.icon }} {{ forceState.label }}
              </span>
              <span class="text-slate-500 text-[10px]">
                {{ forceAnalogy.icon }} {{ forceAnalogy.text }}
              </span>
            </div>
            <p class="text-[11px] leading-relaxed text-slate-400">{{ forceState.desc }}</p>
          </div>

          <!-- 应力排名 -->
          <div class="bg-slate-900 rounded-lg p-2.5">
            <div class="flex items-center justify-between mb-1">
              <span class="font-bold text-[11px]" :style="{ color: stressRank.color }">
                {{ stressRank.level }}
              </span>
              <span class="font-mono text-slate-500 text-[10px]">
                占最大值 {{ (stressRatioToMax * 100).toFixed(0) }}%
              </span>
            </div>
            <p class="text-[11px] leading-relaxed text-slate-400">{{ stressRank.desc }}</p>
          </div>

          <!-- 变形解读 -->
          <div class="bg-slate-900 rounded-lg p-2.5">
            <div class="flex items-center justify-between mb-1">
              <span class="font-bold text-[11px] text-purple-400">
                📏 变形：{{ deformationAnalogy.level }}
              </span>
              <span class="font-mono text-slate-500 text-[10px]">
                {{ strainPerMille.toFixed(3) }}‰
              </span>
            </div>
            <p class="text-[11px] leading-relaxed text-slate-400">
              绝对变形量 <span class="text-slate-300 font-mono">{{ absoluteDeformationMM.toFixed(4) }} mm</span>，
              {{ deformationAnalogy.text }}
            </p>
          </div>
        </div>
      </div>

      <!-- ─── 风险评估与建议 ─────────────────────────────────────────────── -->
      <div v-if="hasResult" class="border-t border-slate-700 pt-2">
        <div class="text-slate-400 mb-2 flex items-center gap-1">
          <span>🛡️</span> 安全评估与建议
        </div>

        <!-- 利用度进度条 -->
        <div class="bg-slate-900 rounded-lg p-2.5 mb-2">
          <div class="flex items-center justify-between mb-2">
            <span class="text-slate-400 text-[11px]">材料利用度</span>
            <span class="font-mono text-[11px] font-bold" :style="{ color: risk.color }">
              {{ (utilization * 100).toFixed(1) }}%
            </span>
          </div>
          <div class="h-3 bg-slate-700 rounded-full overflow-hidden relative">
            <div
              class="h-full rounded-full transition-all duration-500"
              :style="{ width: utilizationPct + '%', backgroundColor: risk.color }"
            />
            <div class="absolute inset-0 flex items-center justify-between px-1 pointer-events-none">
              <div class="w-0.5 h-2 bg-slate-500/50" style="left: 50%"></div>
            </div>
          </div>
          <div class="flex justify-between mt-1.5 text-[9px] text-slate-500">
            <span>0 闲置</span>
            <span class="text-green-500/70">50%</span>
            <span class="text-yellow-500/70">85%</span>
            <span class="text-red-500/70">100% 极限</span>
          </div>
        </div>

        <!-- 风险说明卡片 -->
        <div
          class="rounded-lg p-2.5 mb-2 border-l-4"
          :style="{ backgroundColor: risk.bg, borderColor: risk.color }"
        >
          <div class="flex items-center justify-between mb-1.5">
            <div class="flex items-center gap-1.5">
              <span class="text-base">{{ risk.emoji }}</span>
              <span class="font-bold text-[12px]" :style="{ color: risk.color }">
                风险等级：{{ risk.level }}
              </span>
            </div>
            <div class="text-right">
              <div class="text-[9px] text-slate-500">安全系数</div>
              <div class="font-mono font-bold text-[11px]" :style="{ color: risk.color }">
                {{ safetyFactor === Infinity ? '∞' : safetyFactor.toFixed(2) }}
              </div>
            </div>
          </div>
          <p class="text-[11px] leading-relaxed text-slate-300">{{ risk.desc }}</p>
          <div class="mt-2 pt-2 border-t border-slate-700/50">
            <div class="text-[10px] text-slate-500 mb-1">📋 建议措施：</div>
            <ul class="space-y-0.5">
              <li
                v-for="(tip, idx) in risk.advice"
                :key="idx"
                class="text-[10px] leading-relaxed text-slate-400 flex items-start gap-1"
              >
                <span>•</span>
                <span>{{ tip }}</span>
              </li>
            </ul>
          </div>
        </div>

        <!-- 受压杆件失稳专项提示 -->
        <div
          v-if="bucklingRisk"
          class="rounded-lg p-2.5 border-l-4"
          :style="{
            backgroundColor: bucklingRisk.color + '15',
            borderColor: bucklingRisk.color,
          }"
        >
          <div class="flex items-center justify-between mb-1">
            <span class="font-bold text-[11px]" :style="{ color: bucklingRisk.color }">
              🏗️ 受压失稳风险：{{ bucklingRisk.level }}
            </span>
            <span class="font-mono text-[10px] text-slate-500">
              长细比 λ ≈ {{ slendernessRatio.toFixed(0) }}
            </span>
          </div>
          <p class="text-[11px] leading-relaxed text-slate-400">{{ bucklingRisk.desc }}</p>
        </div>

        <!-- 材料参数参考 -->
        <div class="bg-slate-900 rounded-lg p-2.5 mt-2">
          <div class="text-slate-500 text-[10px] mb-1">📐 参考参数</div>
          <div class="grid grid-cols-2 gap-2 text-[11px]">
            <div class="flex justify-between">
              <span class="text-slate-500">屈服强度</span>
              <span class="font-mono text-slate-300">{{ (yieldStrength / 1e6).toFixed(0) }} MPa</span>
            </div>
            <div class="flex justify-between">
              <span class="text-slate-500">当前应力</span>
              <span
                class="font-mono"
                :style="{ color: risk.color }"
              >{{ (Math.abs(selectedEl.stress) / 1e6).toFixed(2) }} MPa</span>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
