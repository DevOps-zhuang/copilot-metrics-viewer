<template>
    <div class="tiles-container">      
        <v-card elevation="4" color="white" variant="elevated" class="mx-auto my-3" style="width: 300px; height: 175px;" @click="filterSeats('total')">
            <v-card-item class="d-flex justify-center align-center">
                <div class="tiles-text">
                    <div class="text-overline mb-1" style="visibility: hidden;">filler</div>
                    <div class="text-h6 mb-1">Total Assigned  </div>
                    <div class="text-caption">
                        Currently assigned seats
                    </div>
                    <p class="text-h4">{{ totalSeats.length }}</p>
                </div>
            </v-card-item>
        </v-card>

        <v-card elevation="4" color="white" variant="elevated" class="mx-auto my-3" style="width: 300px; height: 175px;" @click="filterSeats('noshow')">
            <v-card-item class="d-flex justify-center align-center">
                <div class="tiles-text">
                    <div class="text-overline mb-1" style="visibility: hidden;">filler</div>
                    <div class="text-h6 mb-1">Assigned But Never Used</div>
                    <div class="text-caption">
                        No show seats
                    </div>
                    <p class="text-h4">{{ noShowSeats.length }}</p>
                </div>
            </v-card-item>
        </v-card>
        <v-card elevation="4" color="white" variant="elevated" class="mx-auto my-3" style="width: 300px; height: 175px;" @click="filterSeats('unused')">
            <v-card-item class="d-flex justify-center align-center">
                <div class="tiles-text">
                    <div class="text-overline mb-1" style="visibility: hidden;">filler</div>
                    <div class="text-h6 mb-1">No Activity in the Last 7 days </div>
                    <div class="text-caption">
                        No use in the last 7 days
                    </div>
                    <p class="text-h4">{{ unusedSeats.length }}</p>
                </div>
            </v-card-item>
        </v-card>
        <v-card elevation="4" color="white" variant="elevated" class="mx-auto my-3" style="width: 300px; height: 175px;" @click="filterSeats('noActivity30Days')">
            <v-card-item class="d-flex justify-center align-center">
                <div class="tiles-text">
                    <div class="text-overline mb-1" style="visibility: hidden;">filler</div>
                    <div class="text-h6 mb-1">No Activity in the Last 30 Days</div>
                    <div class="text-caption">
                        No use in the last 30 days
                    </div>
                    <p class="text-h4">{{ noActivity30DaysSeats.length }}</p>
                </div>
            </v-card-item>
        </v-card>
        <v-card elevation="4" color="white" variant="elevated" class="mx-auto my-3" style="width: 300px; height: 175px;" @click="filterSeats('noActivity60Days')">
            <v-card-item class="d-flex justify-center align-center">
                <div class="tiles-text">
                    <div class="text-overline mb-1" style="visibility: hidden;">filler</div>
                    <div class="text-h6 mb-1">No Activity in the Last 60 Days</div>
                    <div class="text-caption">
                        No use in the last 60 days
                    </div>
                    <p class="text-h4">{{ noActivity60DaysSeats.length }}</p>
                </div>
            </v-card-item>
        </v-card>
    </div>
    
    <div>
        <v-main class="p-1" style="min-height: 300px;">
            <v-container style="min-height: 300px;" class="px-4 elevation-2">
                <br>
                <h2>All assigned seats </h2>
                <br>
            <v-data-table :headers="headers" :items="filteredSeats" :items-per-page="10" class="elevation-2">
                <template v-slot:item="{ item, index }">
                    <tr>
                        <td>{{ index + 1 }}</td>
                        <td>{{ item.login }}</td>
                        <td>{{ item.id }}</td>
                        <td>{{ item.team }}</td>
                        <td>{{ item.created_at }}</td>
                        <td>{{ item.last_activity_at }}</td>
                        <td>{{ item.last_activity_editor }}</td>
                    </tr>
                </template>
                </v-data-table>
            </v-container>
        </v-main>
    </div>
</template>
  
<script lang="ts">
  import { defineComponent, ref , watchEffect, onMounted } from 'vue';
  import { Seat } from '../model/Seat';
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
name: 'SeatsAnalysisViewer',
props: {
        seats: {
            type: Array as () => Seat[],
            required: true,
            default: () => []  
        }
    },
data() {
    return {
        headers: [
            { title: 'S.No', key: 'serialNumber'},
            { title: 'Login', key: 'login' },
            { title: 'GitHub ID', key: 'id' },
            { title: 'Assigning team', key: 'team' },
            { title: 'Assigned time', key: 'created_at' },
            { title: 'Last Activity At', key: 'last_activity_at' },
            { title: 'Last Activity Editor', key: 'last_activity_editor' },
        ],
    };
},
setup(props) {
    let totalSeats = ref<Seat[]>([]);
    let noShowSeats = ref<Seat[]>([]);
    const unusedSeats = ref<Seat[]>([]);
    const noActivity30DaysSeats = ref<Seat[]>([]);
    const noActivity60DaysSeats = ref<Seat[]>([]);
    const filteredSeats = ref<Seat[]>([]);

    watchEffect(() => {
        if (props.seats && Array.isArray(props.seats)) {
            totalSeats.value = props.seats;

            const oneWeekAgo = new Date();
            oneWeekAgo.setDate(oneWeekAgo.getDate() - 7);

            const thirtyDaysAgo = new Date();
            thirtyDaysAgo.setDate(thirtyDaysAgo.getDate() - 30);

            const sixtyDaysAgo = new Date();
            sixtyDaysAgo.setDate(sixtyDaysAgo.getDate() - 60);

            noShowSeats.value = props.seats.filter(seat => seat.last_activity_at == null);

            unusedSeats.value = totalSeats.value.filter(seat => {
                if (seat.last_activity_at === null) {
                    return true; // consider including the last activity date if it is null in the unused seats
                }

                const lastActivityDate = new Date(seat.last_activity_at);
                return lastActivityDate < oneWeekAgo;
            });

            noActivity30DaysSeats.value = totalSeats.value.filter(seat => {
                if (seat.last_activity_at === null) {
                    return true;
                }

                const lastActivityDate = new Date(seat.last_activity_at);
                return lastActivityDate < thirtyDaysAgo;
            });

            noActivity60DaysSeats.value = totalSeats.value.filter(seat => {
                if (seat.last_activity_at === null) {
                    return true;
                }

                const lastActivityDate = new Date(seat.last_activity_at);
                return lastActivityDate < sixtyDaysAgo;
            });

            // to sort  unusedSeats by last_activity_at
            unusedSeats.value.sort((a, b) => {
                if (a.last_activity_at === null) {
                    return -1;
                }
                if (b.last_activity_at === null) {
                    return 1;
                }
                return new Date(a.last_activity_at) > new Date(b.last_activity_at) ? 1 : -1;
            });

            // to sort noActivity30DaysSeats by last_activity_at
            noActivity30DaysSeats.value.sort((a, b) => {
                if (a.last_activity_at === null) {
                    return -1;
                }
                if (b.last_activity_at === null) {
                    return 1;
                }
                return new Date(a.last_activity_at) > new Date(b.last_activity_at) ? 1 : -1;
            });

            // to sort noActivity60DaysSeats by last_activity_at
            noActivity60DaysSeats.value.sort((a, b) => {
                if (a.last_activity_at === null) {
                    return -1;
                }
                if (b.last_activity_at === null) {
                    return 1;
                }
                return new Date(a.last_activity_at) > new Date(b.last_activity_at) ? 1 : -1;
            });
        } else {
            throw new Error('Invalid number of seats');
        }

    });

    const filterSeats = (type: string) => {
        if (type === 'total') {
            filteredSeats.value = totalSeats.value;
        } else if (type === 'noshow') {
            filteredSeats.value = noShowSeats.value;
        } else if (type === 'unused') {
            filteredSeats.value = unusedSeats.value;
        } else if (type === 'noActivity30Days') {
            filteredSeats.value = noActivity30DaysSeats.value;
        } else if (type === 'noActivity60Days') {
            filteredSeats.value = noActivity60DaysSeats.value;
        }
    };

    onMounted(() => {
        filterSeats('total'); // 默认选择 "Total Assigned" 框
    });

    return {
        totalSeats,
        noShowSeats,
        unusedSeats,
        noActivity30DaysSeats,
        noActivity60DaysSeats,
        filterSeats,
        filteredSeats
    }
}   
  
});
</script>
