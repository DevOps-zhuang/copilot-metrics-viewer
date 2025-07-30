<template>
  <div>
    <div class="tiles-container">
      <!-- Acceptance Rate Tile -->  
      <v-card elevation="4" color="white" variant="elevated" class="mx-auto my-3" style="width: 300px; height: 175px;">
          <v-card-item>
            <div class="tiles-text">
              <div class="spacing-25"/>
              <div class="text-h6 mb-1">Number of {{ breakdownDisplayNamePlural }}</div>
              <div class="text-caption">
                Over the last 28 days
              </div>
              <p class="text-h4">{{ numberOfBreakdowns }}</p> 
          </div>
        </v-card-item>
      </v-card>
    </div>

    <v-main class="p-1" style="min-height: 300px;">
      <v-container style="min-height: 300px;" class="px-4 elevation-2">
        <v-row>
          <v-col cols="4">
            <v-card>
              <v-card-item class="d-flex justify-center align-center">
                <div class="spacing-25"/>
                <div class="text-h6 mb-1">Top 5 {{ breakdownDisplayNamePlural }} by accepted suggestions (prompts)</div>
                <div style="width: 300px; height: 300px;">
                  <Pie :data="breakdownsChartDataTop5AcceptedPrompts" :options="chartOptions" />
                </div>
              </v-card-item>
            </v-card>
          </v-col>

          <v-col cols="4">
            <v-card>
              <v-card-item class="d-flex justify-center align-center">
                <div class="spacing-25"/>
                <div class="text-h6 mb-1">Acceptance Rate (by count) for Top 5 {{ breakdownDisplayNamePlural }}</div>
                <div style="width: 300px; height: 300px;">
                  <Pie :data="breakdownsChartDataTop5AcceptedPromptsByCounts" :options="chartOptions" />
                </div>
              </v-card-item>
            </v-card>
          </v-col>

          <v-col cols="4">
            <v-card>
              <v-card-item class="d-flex justify-center align-center">
                <div class="spacing-25"/>
                <div class="text-h6 mb-1">Acceptance Rate (by code lines) for Top 5 {{ breakdownDisplayNamePlural }}</div>
                <div style="width: 300px; height: 300px;">
                  <Pie :data="breakdownsChartDataTop5AcceptedPromptsByLines" :options="chartOptions" />
                </div>
              </v-card-item>
            </v-card>
          </v-col>
        </v-row>

        <br>
        <h2>{{ breakdownDisplayNamePlural }} Breakdown </h2>
        <br>

        <v-data-table :headers="headers" :items="breakdownList" class="elevation-2" style="padding-left: 100px; padding-right: 100px;">
            <template #item="{item}">
                <tr>
                    <td>
                      <div style="display: flex; align-items: center;">
                        <v-btn 
                          v-if="$props.childKey && item.children && item.children.length > 0"
                          icon
                          size="small"
                          variant="text"
                          @click="toggleExpanded(item.name)"
                          style="margin-right: 8px;"
                        >
                          <v-icon>{{ item.isExpanded ? 'mdi-chevron-down' : 'mdi-chevron-right' }}</v-icon>
                        </v-btn>
                        <span style="margin-left: 8px;">{{ item.name }}</span>
                      </div>
                    </td>
                    <td>{{ item.acceptedPrompts }}</td>
                    <td>{{ item.suggestedPrompts }}</td>
                    <td>{{ item.acceptedLinesOfCode }}</td>
                    <td>{{ item.suggestedLinesOfCode }}</td>
                    <td v-if="item.acceptanceRateByCount !== undefined">{{ item.acceptanceRateByCount.toFixed(2) }}%</td>
                    <td v-if="item.acceptanceRateByLines !== undefined">{{ item.acceptanceRateByLines.toFixed(2) }}%</td>
                </tr>
                <!-- 子项显示 -->
                <template v-if="$props.childKey && item.children && item.children.length > 0 && item.isExpanded">
                  <tr v-for="child in item.children" :key="`${item.name}-${child.name}`" class="child-row">
                    <td style="padding-left: 40px;">
                      <span style="color: #666;">{{ child.name }}</span>
                    </td>
                    <td>{{ child.acceptedPrompts }}</td>
                    <td>{{ child.suggestedPrompts }}</td>
                    <td>{{ child.acceptedLinesOfCode }}</td>
                    <td>{{ child.suggestedLinesOfCode }}</td>
                    <td v-if="child.acceptanceRateByCount !== undefined">{{ child.acceptanceRateByCount.toFixed(2) }}%</td>
                    <td v-if="child.acceptanceRateByLines !== undefined">{{ child.acceptanceRateByLines.toFixed(2) }}%</td>
                  </tr>
                </template>
            </template>
        </v-data-table>
      </v-container>
    </v-main>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, toRef, watch, computed } from 'vue';
import type { Metrics } from '@/model/Metrics';
import { Breakdown } from '@/model/Breakdown';
import { Pie } from 'vue-chartjs'

import {
  Chart as ChartJS,
  ArcElement,
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  BarElement,
  Title,
  Tooltip,
  Legend
} from 'chart.js'

ChartJS.register(
  ArcElement, 
  CategoryScale,
  LinearScale,
  BarElement,
  PointElement,
  LineElement,
  Title,
  Tooltip,
  Legend
)

export default defineComponent({
  name: 'BreakdownComponent',
  components: {
    Pie
  },
  props: {
      metrics: {
          type: Object,
          required: true
      },
      breakdownKey: {
          type: String,
          required: true
      },
      childKey: {
          type: String,
          required: false
      }
  },
  setup(props: { metrics: any; breakdownKey: string; childKey?: string }) {
    // Create a reactive reference to store the breakdowns.
    const breakdownList = ref<Breakdown[]>([]);
    const expandedItems = ref<string[]>([]);

    // Number of breakdowns
    const numberOfBreakdowns = ref(0);

    // Breakdowns Chart Data for breakdowns breakdown Pie Chart
    const breakdownsChartData = ref<{ labels: string[]; datasets: any[] }>({ labels: [], datasets: [] });

    //Top 5 by accepted prompts
    const breakdownsChartDataTop5AcceptedPrompts = ref<{ labels: string[]; datasets: any[] }>({ labels: [], datasets: [] });

    //Acceptance Rate by lines for top 5 by accepted prompts
    const breakdownsChartDataTop5AcceptedPromptsByLines = ref<{ labels: string[]; datasets: any[] }>({ labels: [], datasets: [] });

    //Acceptance Rate by counts for top 5 by accepted prompts
    const breakdownsChartDataTop5AcceptedPromptsByCounts = ref<{ labels: string[]; datasets: any[] }>({ labels: [], datasets: [] });

    const chartOptions = {
      responsive: true,
      maintainAspectRatio: true,
    };

    const pieChartColors = ref([
    '#4B0082', // Indigo
    '#41B883', // Vue Green
    '#6495ED', // Cornflower Blue
    '#87CEFA', // Light Sky Blue
    '#7CFC00'  // Lawn Green
    ]);

    // 计算嵌套数据显示列表，包含展开/折叠状态
    const displayBreakdownList = computed(() => {
      if (!props.childKey) {
        return breakdownList.value;
      }
      
      return breakdownList.value.map((item: Breakdown) => ({
        ...item,
        children: getChildrenForBreakdown(item),
        isExpanded: expandedItems.value.includes(item.name)
      }));
    });

    // 获取指定 breakdown 的子项
    function getChildrenForBreakdown(parentBreakdown: Breakdown): Breakdown[] {
      if (!props.childKey) return [];
      
      const children: Breakdown[] = [];
      
      // 遍历原始 metrics 数据，找到属于该父项的子项
      props.metrics.forEach((m: any) => {
        m.breakdown.forEach((breakdownData: any) => {
          const parentName = breakdownData[props.breakdownKey];
          if (parentName === parentBreakdown.name) {
            const childName = breakdownData[props.childKey!];
            let child = children.find(c => c.name === childName);
            
            if (!child) {
              child = new Breakdown({
                name: childName,
                acceptedPrompts: breakdownData.acceptances_count || 0,
                suggestedPrompts: breakdownData.suggestions_count || 0,
                suggestedLinesOfCode: breakdownData.lines_suggested || 0,
                acceptedLinesOfCode: breakdownData.lines_accepted || 0,
              });
              children.push(child);
            } else {
              child.acceptedPrompts += breakdownData.acceptances_count || 0;
              child.suggestedPrompts += breakdownData.suggestions_count || 0;
              child.suggestedLinesOfCode += breakdownData.lines_suggested || 0;
              child.acceptedLinesOfCode += breakdownData.lines_accepted || 0;
            }
            
            // 重新计算接受率
            child.acceptanceRateByCount = child.suggestedPrompts !== 0 ? (child.acceptedPrompts / child.suggestedPrompts) * 100 : 0;
            child.acceptanceRateByLines = child.suggestedLinesOfCode !== 0 ? (child.acceptedLinesOfCode / child.suggestedLinesOfCode) * 100 : 0;
          }
        });
      });
      
      return children.sort((a, b) => b.acceptedPrompts - a.acceptedPrompts);
    }

    // 切换展开/折叠状态
    function toggleExpanded(itemName: string) {
      const index = expandedItems.value.indexOf(itemName);
      if (index > -1) {
        expandedItems.value.splice(index, 1);
      } else {
        expandedItems.value.push(itemName);
      }
    };

    // 使用 watch 监听 props.metrics 的变化
    watch(() => props.metrics, (newMetrics: any) => {
      console.log('BreakdownComponent received new metrics data:', newMetrics.length);
      // 强制创建一个新的引用，确保数据变化被检测到
      const metricsCopy = Array.isArray(newMetrics) ? [...newMetrics] : [newMetrics];
      processBreakdownData(metricsCopy);
    }, { immediate: true, deep: true });

    // 将数据处理逻辑封装到单独的函数中
    function processBreakdownData(data: any[]) {
      // 清空现有数据
      breakdownList.value = [];

      // 处理分解数据
      data.forEach((m: any) => m.breakdown.forEach((breakdownData: any) => {
        const breakdownName = breakdownData[props.breakdownKey as keyof typeof breakdownData] as string;
        let breakdown = breakdownList.value.find((b: Breakdown) => b.name === breakdownName);

        if (!breakdown) {
          // 如果不存在则创建新的分解对象
          breakdown = new Breakdown({
            name: breakdownName,
            acceptedPrompts: breakdownData.acceptances_count,
            suggestedPrompts: breakdownData.suggestions_count,
            suggestedLinesOfCode: breakdownData.lines_suggested,
            acceptedLinesOfCode: breakdownData.lines_accepted,
          });
          breakdownList.value.push(breakdown);
        } else {
          // 更新现有分解对象
          breakdown.acceptedPrompts += breakdownData.acceptances_count;
          breakdown.suggestedPrompts += breakdownData.suggestions_count;
          breakdown.suggestedLinesOfCode += breakdownData.lines_suggested;
          breakdown.acceptedLinesOfCode += breakdownData.lines_accepted;
        }
        // 重新计算接受率
        breakdown.acceptanceRateByCount = breakdown.suggestedPrompts !== 0 ? (breakdown.acceptedPrompts / breakdown.suggestedPrompts) * 100 : 0;
        breakdown.acceptanceRateByLines = breakdown.suggestedLinesOfCode !== 0 ? (breakdown.acceptedLinesOfCode / breakdown.suggestedLinesOfCode) * 100 : 0;
      }));

      // 按接受的提示数量对分解列表进行排序
      breakdownList.value.sort((a: Breakdown, b: Breakdown) => b.acceptedPrompts - a.acceptedPrompts);

      // 获取接受的提示数量排名前 5 的分解
      const top5BreakdownsAcceptedPrompts = breakdownList.value.slice(0, 5);
      numberOfBreakdowns.value = breakdownList.value.length;

      // 更新图表数据
      breakdownsChartDataTop5AcceptedPrompts.value = {
        labels: top5BreakdownsAcceptedPrompts.map((breakdown: Breakdown) => breakdown.name),
        datasets: [
          {
            data: top5BreakdownsAcceptedPrompts.map((breakdown: Breakdown) => breakdown.acceptedPrompts),
            backgroundColor: pieChartColors.value,
          }
        ]
      };

      // 更新前 5 个分解的接受率 (按代码行)
      breakdownsChartDataTop5AcceptedPromptsByLines.value = {
        labels: top5BreakdownsAcceptedPrompts.map((breakdown: Breakdown) => breakdown.name),
        datasets: [
          {
            data: top5BreakdownsAcceptedPrompts.map((breakdown: Breakdown) => breakdown.acceptanceRateByLines),
            backgroundColor: pieChartColors.value,
          }
        ]
      };

      // 更新前 5 个分解的接受率 (按计数)
      breakdownsChartDataTop5AcceptedPromptsByCounts.value = {
        labels: top5BreakdownsAcceptedPrompts.map((breakdown: Breakdown) => breakdown.name),
        datasets: [
          {
            data: top5BreakdownsAcceptedPrompts.map((breakdown: Breakdown) => breakdown.acceptanceRateByCount),
            backgroundColor: pieChartColors.value,
          }
        ]
      };
    }

    // 计算属性
    const breakdownDisplayName = computed(() => {
      return props.breakdownKey.charAt(0).toUpperCase() + props.breakdownKey.slice(1);
    });

    const breakdownDisplayNamePlural = computed(() => {
      return `${breakdownDisplayName.value}s`;
    });

    const headers = computed(() => {
      return [
        { title: `${breakdownDisplayName.value} Name`, key: 'name' },
        { title: 'Accepted Prompts', key: 'acceptedPrompts' },
        { title: 'Suggested Prompts', key: 'suggestedPrompts' },
        { title: 'Accepted Lines of Code', key: 'acceptedLinesOfCode' },
        { title: 'Suggested Lines of Code', key: 'suggestedLinesOfCode' },
        { title: 'Acceptance Rate by Count (%)', key: 'acceptanceRateByCount' },
        { title: 'Acceptance Rate by Lines (%)', key: 'acceptanceRateByLines' },
      ];
    });

    return { 
      chartOptions, 
      breakdownList: displayBreakdownList,
      numberOfBreakdowns, 
      breakdownsChartData, 
      breakdownsChartDataTop5AcceptedPrompts, 
      breakdownsChartDataTop5AcceptedPromptsByLines, 
      breakdownsChartDataTop5AcceptedPromptsByCounts,
      toggleExpanded,
      breakdownDisplayName,
      breakdownDisplayNamePlural,
      headers
    };
  }
});
</script>

<style scoped>
.child-row {
  background-color: #f9f9f9;
}

.child-row td {
  border-top: 1px solid #e0e0e0;
}
</style>
