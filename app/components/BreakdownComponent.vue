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
                      <div class="d-flex align-center">
                        <v-btn 
                          v-if="breakdownKey === 'language' && item.editors && item.editors.length > 0"
                          icon
                          size="small"
                          variant="text"
                          @click="toggleExpansion(item.name)"
                          class="mr-2"
                        >
                          <v-icon>{{ expandedItems.has(item.name) ? 'mdi-chevron-down' : 'mdi-chevron-right' }}</v-icon>
                        </v-btn>
                        <span>{{ item.name }}</span>
                      </div>
                    </td>
                    <td>{{ item.acceptedPrompts }}</td>
                    <td>{{ item.suggestedPrompts }}</td>
                    <td>{{ item.acceptedLinesOfCode }}</td>
                    <td>{{ item.suggestedLinesOfCode }}</td>
                    <td v-if="item.acceptanceRateByCount !== undefined">{{ item.acceptanceRateByCount.toFixed(2) }}%</td>
                    <td v-if="item.acceptanceRateByLines !== undefined">{{ item.acceptanceRateByLines.toFixed(2) }}%</td>
                </tr>
                <!-- Editor breakdown rows for expanded languages -->
                <template v-if="breakdownKey === 'language' && item.editors && expandedItems.has(item.name)">
                  <tr v-for="editor in item.editors" :key="`${item.name}-${editor.name}`" class="editor-row">
                    <td class="pl-10">
                      <v-icon class="mr-2" size="small">mdi-subdirectory-arrow-right</v-icon>
                      {{ editor.name }}
                    </td>
                    <td>{{ editor.acceptedPrompts }}</td>
                    <td>{{ editor.suggestedPrompts }}</td>
                    <td>{{ editor.acceptedLinesOfCode }}</td>
                    <td>{{ editor.suggestedLinesOfCode }}</td>
                    <td>{{ editor.acceptanceRateByCount.toFixed(2) }}%</td>
                    <td>{{ editor.acceptanceRateByLines.toFixed(2) }}%</td>
                  </tr>
                </template>
            </template>
        </v-data-table>
      </v-container>
    </v-main>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, toRef } from 'vue';
import type { Metrics } from '@/model/Metrics';
import { Breakdown, BreakdownEditor } from '@/model/Breakdown';
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
      }
  },
  setup(props) {

    // Create a reactive reference to store the breakdowns.
    const breakdownList = ref<Breakdown[]>([]);

    // Track expanded items for nested breakdown
    const expandedItems = ref<Set<string>>(new Set());
    
    // Toggle expansion of a breakdown item
    const toggleExpansion = (itemName: string) => {
      if (expandedItems.value.has(itemName)) {
        expandedItems.value.delete(itemName);
      } else {
        expandedItems.value.add(itemName);
      }
    };

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

    const data = toRef(props, 'metrics').value;

    // Process the breakdown separately
    if (props.breakdownKey === 'language') {
      // For language breakdown, create nested structure with editors
      const languageMap = new Map<string, Map<string, { acceptedPrompts: number, suggestedPrompts: number, suggestedLinesOfCode: number, acceptedLinesOfCode: number }>>();
      
      data.forEach((m: Metrics) => m.breakdown.forEach(breakdownData => {
        const language = breakdownData.language;
        const editor = breakdownData.editor;
        
        if (!languageMap.has(language)) {
          languageMap.set(language, new Map());
        }
        
        const editorsMap = languageMap.get(language)!;
        if (!editorsMap.has(editor)) {
          editorsMap.set(editor, {
            acceptedPrompts: 0,
            suggestedPrompts: 0,
            suggestedLinesOfCode: 0,
            acceptedLinesOfCode: 0
          });
        }
        
        const editorData = editorsMap.get(editor)!;
        editorData.acceptedPrompts += breakdownData.acceptances_count;
        editorData.suggestedPrompts += breakdownData.suggestions_count;
        editorData.suggestedLinesOfCode += breakdownData.lines_suggested;
        editorData.acceptedLinesOfCode += breakdownData.lines_accepted;
      }));
      
      // Convert to breakdown list with nested editor information
      for (const [language, editorsMap] of languageMap) {
        const editors: BreakdownEditor[] = [];
        let totalAcceptedPrompts = 0;
        let totalSuggestedPrompts = 0;
        let totalSuggestedLinesOfCode = 0;
        let totalAcceptedLinesOfCode = 0;
        
        for (const [editorName, editorData] of editorsMap) {
          const acceptanceRateByCount = editorData.suggestedPrompts !== 0 ? (editorData.acceptedPrompts / editorData.suggestedPrompts) * 100 : 0;
          const acceptanceRateByLines = editorData.suggestedLinesOfCode !== 0 ? (editorData.acceptedLinesOfCode / editorData.suggestedLinesOfCode) * 100 : 0;
          
          editors.push(new BreakdownEditor({
            name: editorName,
            acceptedPrompts: editorData.acceptedPrompts,
            suggestedPrompts: editorData.suggestedPrompts,
            suggestedLinesOfCode: editorData.suggestedLinesOfCode,
            acceptedLinesOfCode: editorData.acceptedLinesOfCode,
            acceptanceRateByCount,
            acceptanceRateByLines
          }));
          
          totalAcceptedPrompts += editorData.acceptedPrompts;
          totalSuggestedPrompts += editorData.suggestedPrompts;
          totalSuggestedLinesOfCode += editorData.suggestedLinesOfCode;
          totalAcceptedLinesOfCode += editorData.acceptedLinesOfCode;
        }
        
        // Sort editors by accepted prompts
        editors.sort((a, b) => b.acceptedPrompts - a.acceptedPrompts);
        
        const breakdown = new Breakdown({
          name: language,
          acceptedPrompts: totalAcceptedPrompts,
          suggestedPrompts: totalSuggestedPrompts,
          suggestedLinesOfCode: totalSuggestedLinesOfCode,
          acceptedLinesOfCode: totalAcceptedLinesOfCode,
          acceptanceRateByCount: totalSuggestedPrompts !== 0 ? (totalAcceptedPrompts / totalSuggestedPrompts) * 100 : 0,
          acceptanceRateByLines: totalSuggestedLinesOfCode !== 0 ? (totalAcceptedLinesOfCode / totalSuggestedLinesOfCode) * 100 : 0,
          editors
        });
        breakdownList.value.push(breakdown);
      }
    } else {
      // Original logic for editor breakdown and other breakdown types
      data.forEach((m: Metrics) => m.breakdown.forEach(breakdownData => 
      {
        const breakdownName = breakdownData[props.breakdownKey as keyof typeof breakdownData] as string;
        let breakdown = breakdownList.value.find(b => b.name === breakdownName);

        if (!breakdown) {
          // Create a new breakdown object if it does not exist
          breakdown = new Breakdown({
            name: breakdownName,
            acceptedPrompts: breakdownData.acceptances_count,
            suggestedPrompts: breakdownData.suggestions_count,
            suggestedLinesOfCode: breakdownData.lines_suggested,
            acceptedLinesOfCode: breakdownData.lines_accepted,
          });
          breakdownList.value.push(breakdown);
        } else {
          // Update the existing breakdown object
          breakdown.acceptedPrompts += breakdownData.acceptances_count;
          breakdown.suggestedPrompts += breakdownData.suggestions_count;
          breakdown.suggestedLinesOfCode += breakdownData.lines_suggested;
          breakdown.acceptedLinesOfCode += breakdownData.lines_accepted;
        }
        // Recalculate the acceptance rates
        breakdown.acceptanceRateByCount = breakdown.suggestedPrompts !== 0 ? (breakdown.acceptedPrompts / breakdown.suggestedPrompts) * 100 : 0;
        breakdown.acceptanceRateByLines = breakdown.suggestedLinesOfCode !== 0 ? (breakdown.acceptedLinesOfCode / breakdown.suggestedLinesOfCode) * 100 : 0;

        // Log each breakdown for debugging
       // console.log('Breakdown:', breakdown);
      }));
    }

    //Sort breakdowns map by accepted prompts
    breakdownList.value.sort((a, b) => b.acceptedPrompts - a.acceptedPrompts);

    // Get the top 5 breakdowns by accepted prompts
    const top5BreakdownsAcceptedPrompts = breakdownList.value.slice(0, 5);
    
    breakdownsChartDataTop5AcceptedPrompts.value = {
      labels: top5BreakdownsAcceptedPrompts.map(breakdown => breakdown.name),
      datasets: [
        {
          data: top5BreakdownsAcceptedPrompts.map(breakdown => breakdown.acceptedPrompts),
          backgroundColor: pieChartColors.value,
        },
      ],
    };

    breakdownsChartDataTop5AcceptedPromptsByLines.value = {
      labels: top5BreakdownsAcceptedPrompts.map(breakdown => breakdown.name),
      datasets: [
        {
          data: top5BreakdownsAcceptedPrompts.map(breakdown => breakdown.acceptanceRateByLines.toFixed(2)),
          backgroundColor: pieChartColors.value,
        },
      ],
    };

    breakdownsChartDataTop5AcceptedPromptsByCounts.value = {
      labels: top5BreakdownsAcceptedPrompts.map(breakdown => breakdown.name),
      datasets: [
        {
          data: top5BreakdownsAcceptedPrompts.map(breakdown => breakdown.acceptanceRateByCount.toFixed(2)),
          backgroundColor: pieChartColors.value,
        },
      ],
    };

    numberOfBreakdowns.value = breakdownList.value.length;

    return { chartOptions, breakdownList, numberOfBreakdowns, 
      breakdownsChartData, breakdownsChartDataTop5AcceptedPrompts, breakdownsChartDataTop5AcceptedPromptsByLines, breakdownsChartDataTop5AcceptedPromptsByCounts, expandedItems, toggleExpansion };
  },
  computed: {
    breakdownDisplayName() {
      return this.breakdownKey.charAt(0).toUpperCase() + this.breakdownKey.slice(1);
    },
    breakdownDisplayNamePlural() {
      return `${this.breakdownDisplayName}s`;
    },
    headers() {
      return [
        { title: `${this.breakdownDisplayName} Name`, key: 'name' },
        { title: 'Accepted Prompts', key: 'acceptedPrompts' },
        { title: 'Suggested Prompts', key: 'suggestedPrompts' },
        { title: 'Accepted Lines of Code', key: 'acceptedLinesOfCode' },
        { title: 'Suggested Lines of Code', key: 'suggestedLinesOfCode' },
        { title: 'Acceptance Rate by Count (%)', key: 'acceptanceRateByCount' },
        { title: 'Acceptance Rate by Lines (%)', key: 'acceptanceRateByLines' },
      ];
    },
  },
  

});
</script>

<style scoped>
.editor-row {
  background-color: #f5f5f5;
}

.editor-row td {
  font-size: 0.9em;
  color: #666;
}
</style>
