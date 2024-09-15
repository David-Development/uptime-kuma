<template>
    <div v-if="loadedTheme" class="container mt-3">

        <div v-if="editMode">
            <form>
                <textarea
                    v-model="flowChartTemplate"
                    class="form-control flow-chart-template"
                    autocomplete="off"
                />
            </form>

            <button class="btn btn-info me-2" @click="updateChart()">
                {{ $t("Run") }}
            </button>
            <button class="btn btn-info me-2" @click="editMode = false">
                {{ $t("Close") }}
            </button>
        </div>

        <button class="btn btn-info me-2" data-testid="edit-button" @click="editMode = true">
            <font-awesome-icon icon="edit" />
            {{ $t("Edit") }}
        </button>


        <!-- Main Status Page -->
        <div class="main">
            <!-- Maintenance -->
            <template v-if="maintenanceList.length > 0">
                <div
                    v-for="maintenance in maintenanceList" :key="maintenance.id"
                    class="shadow-box alert mb-4 p-3 bg-maintenance mt-4 position-relative" role="alert"
                >
                    <h4 class="alert-heading">{{ maintenance.title }}</h4>
                    <!-- eslint-disable-next-line vue/no-v-html-->
                    <div class="content" v-html="maintenanceHTML(maintenance.description)"></div>
                    <MaintenanceTime :maintenance="maintenance" />
                </div>
            </template>


            <div class="mb-4">
                <div v-if="$root.publicGroupList.length === 0 && loadedData" class="text-center">
                    <!-- 👀 Nothing here, please add a group or a monitor. -->
                    👀 {{ $t("statusPageNothing") }}
                </div>

                <!--<div id="mermaid-map">{{ $root.flowChartSVG }}</div>-->
                <div id="mermaid-map" class="mermaid">
                    {{ $root.flowChart }}
                </div>
            </div>

            <footer class="mt-5 mb-4">
                <div class="refresh-info mb-2">
                    <div>{{ $t("Last Updated") }}:  {{ lastUpdateTimeDisplay }}</div>
                    <div data-testid="update-countdown-text">{{ $tc("statusPageRefreshIn", [updateCountdownText]) }}</div>
                </div>
            </footer>
        </div>

        <component is="style" v-if="config.customCSS" type="text/css">
            {{ config.customCSS }}
        </component>
    </div>
</template>

<script>
import axios from "axios";
import dayjs from "dayjs";
import duration from "dayjs/plugin/duration";
import Favico from "favico.js";
// import highlighting library (you can use any library you want just return html string)
import { highlight, languages } from "prismjs/components/prism-core";
import "prismjs/components/prism-css";
import "prismjs/themes/prism-tomorrow.css"; // import syntax highlighting styles
import ImageCropUpload from "vue-image-crop-upload";
import { useToast } from "vue-toastification";
import { marked } from "marked";
import DOMPurify from "dompurify";
import Confirm from "../components/Confirm.vue";
import PublicGroupList from "../components/PublicGroupList.vue";
import MaintenanceTime from "../components/MaintenanceTime.vue";
import { getResBaseURL } from "../util-frontend";
import { STATUS_PAGE_ALL_DOWN, STATUS_PAGE_ALL_UP, STATUS_PAGE_MAINTENANCE, STATUS_PAGE_PARTIAL_DOWN, UP, MAINTENANCE } from "../util.ts";
import Tag from "../components/Tag.vue";
import {nextTick} from 'vue'


import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs';
mermaid.initialize({ 
    // https://mermaid.js.org/config/theming.html
    theme: 'neutral',
    startOnLoad: false,
    securityLevel: 'loose',
});


const toast = useToast();
dayjs.extend(duration);

const leavePageMsg = "Do you really want to leave? you have unsaved changes!";

// eslint-disable-next-line no-unused-vars
let feedInterval;

const favicon = new Favico({
    animation: "none"
});

const flowChartTemplateRaw = `
FLOW_CHART_TEMPLATE_PLACEHOLDER
`;

export default {

    components: {
        PublicGroupList,
        MaintenanceTime,
        Tag,
    },

    props: {
        /** Override for the status page slug */
        overrideSlug: {
            type: String,
            required: false,
            default: null,
        },
    },

    data() {
        return {
            slug: null,
            hasToken: false,
            config: {},
            editMode: false,
            imgDataUrl: "/icon.svg",
            loadedTheme: false,
            loadedData: false,
            baseURL: "",
            maintenanceList: [],
            lastUpdateTime: dayjs(),
            updateCountdown: null,
            updateCountdownText: null,
            loading: true,
            flowChart: "",
            flowChartTemplate: flowChartTemplateRaw,
            // flowChartSVG: null,
        };
    },
    computed: {
        maintenanceClass() {
            return "bg-maintenance";
        },

        isMaintenance() {
            return this.overallStatus === STATUS_PAGE_MAINTENANCE;
        },

        lastUpdateTimeDisplay() {
            return this.$root.datetime(this.lastUpdateTime);
        }
    },
    watch: {
        // Set Theme
        "config.theme"() {
            this.$root.statusPageTheme = this.config.theme;
            this.loadedTheme = true;
        },

        "config.title"(title) {
            document.title = title;
        },

        "$root.monitorList"() {
            let count = Object.keys(this.$root.monitorList).length;

            // Since publicGroupList is getting from public rest api, monitors' tags may not present if showTags = false
            if (count > 0) {
                for (let group of this.$root.publicGroupList) {
                    for (let monitor of group.monitorList) {
                        if (monitor.tags === undefined && this.$root.monitorList[monitor.id]) {
                            monitor.tags = this.$root.monitorList[monitor.id].tags;
                        }
                    }
                }
            }

            this.updateChart();
        }

    },
    async created() {
        this.hasToken = ("token" in this.$root.storage());

        // Special handle for dev
        this.baseURL = getResBaseURL();
    },
    async mounted() {
        this.slug = this.overrideSlug || this.$route.params.slug;

        if (!this.slug) {
            this.slug = "default";
        }

        this.getData().then((res) => {
            this.config = res.data.config;

            if (!this.config.domainNameList) {
                this.config.domainNameList = [];
            }

            if (this.config.icon) {
                this.imgDataUrl = this.config.icon;
            }

            this.maintenanceList = res.data.maintenanceList;
            this.$root.publicGroupList = res.data.publicGroupList;

            this.loading = false;

            // Configure auto-refresh loop
            feedInterval = setInterval(() => {
                this.updateHeartbeatList();
            }, (this.config.autoRefreshInterval + 10) * 1000);

            this.updateUpdateTimer();
        }).catch( function (error) {
            if (error.response.status === 404) {
                location.href = "/page-not-found";
            }
            console.log(error);
        });

        this.updateHeartbeatList();
    },
    methods: {

        /**
         * Get status page data
         * It should be preloaded in window.preloadData
         * @returns {Promise<any>} Status page data
         */
        getData: function () {
            if (window.preloadData) {
                return new Promise(resolve => resolve({
                    data: window.preloadData
                }));
            } else {
                return axios.get("/api/status-page/" + this.slug);
            }
        },

        /**
         * Update the heartbeat list and update favicon if necessary
         * @returns {void}
         */
        updateHeartbeatList() {
            axios.get("/api/status-page/heartbeat/" + this.slug).then((res) => {
                const { heartbeatList, uptimeList } = res.data;

                this.$root.heartbeatList = heartbeatList;
                this.$root.uptimeList = uptimeList;

                const heartbeatIds = Object.keys(heartbeatList);
                const downMonitors = heartbeatIds.reduce((downMonitorsAmount, currentId) => {
                    const monitorHeartbeats = heartbeatList[currentId];
                    const lastHeartbeat = monitorHeartbeats.at(-1);

                    if (lastHeartbeat) {
                        return lastHeartbeat.status === 0 ? downMonitorsAmount + 1 : downMonitorsAmount;
                    } else {
                        return downMonitorsAmount;
                    }
                }, 0);

                favicon.badge(downMonitors);

                this.updateChart();

                this.loadedData = true;
                this.lastUpdateTime = dayjs();
                this.updateUpdateTimer();
            });
        },

        /**
         * Setup timer to display countdown to refresh
         * @returns {void}
         */
        updateUpdateTimer() {
            clearInterval(this.updateCountdown);

            this.updateCountdown = setInterval(() => {
                const countdown = dayjs.duration(this.lastUpdateTime.add(this.config.autoRefreshInterval, "seconds").add(10, "seconds").diff(dayjs()));
                if (countdown.as("seconds") < 0) {
                    clearInterval(this.updateCountdown);
                } else {
                    this.updateCountdownText = countdown.format("mm:ss");
                }
            }, 1000);
        },

        updateChart() {
            let flowChart = this.flowChartTemplate;
            // console.log(`START>${flowChart}<END`);
            for(const monitor of Object.values(this.$root.monitorList)) {
                // console.log(monitor.id, monitor.name, this.$root.publicLastHeartbeatList[monitor.id]?.status);

                let statusClass = 'unknown';
                if (this.$root.publicLastHeartbeatList[monitor.id]?.status === 1) {
                    statusClass = 'online'
                } else if(this.$root.publicLastHeartbeatList[monitor.id]?.status === 0) {
                    statusClass = 'offline'
                }

                /*
                // users can give the monitor a shorter name in Mermaid - thus we need to create a mapping
                const pattern = new RegExp(`(\\w+)\\(.*?${monitor.name}\\)`, "gm");
                const mermaidGroup = pattern.exec(flowChart);
                if (mermaidGroup) {
                    const mermaidName = mermaidGroup[1];
                    
                }
                */

                flowChart = flowChart.replace(`${monitor.name})`, `${monitor.name}):::${statusClass}`)
            }
            console.log(flowChart);

            this.$root.flowChart = flowChart;
            // console.log(this.$root.monitorList[1]);
            nextTick(async () => {
                setTimeout(async () => {
                    console.log("update mermaid");
                    await mermaid.run({ querySelector: '#mermaid-map' });

                    // todo.. add dom events.. https://mermaid.js.org/config/usage.html#example-of-a-marked-renderer
                    document.getElementById('mermaid-map').removeAttribute('data-processed');
                    //const { svg } = await mermaid.render('mermaid-map', flowChart);
                    //this.$root.flowChartSVG = svg;
                }, 1000);
            })
        },

    
        /**
         * Generate sanitized HTML from maintenance description
         * @param {string} description Text to sanitize
         * @returns {string} Sanitized HTML
         */
        maintenanceHTML(description) {
            if (description) {
                return DOMPurify.sanitize(marked(description));
            } else {
                return "";
            }
        },

    }
};
</script>

<style lang="scss" scoped>
@import "../assets/vars.scss";

.overall-status {
    font-weight: bold;
    font-size: 25px;

    .ok {
        color: $primary;
    }

    .warning {
        color: $warning;
    }

    .danger {
        color: $danger;
    }
}

h1 {
    font-size: 30px;

    img {
        vertical-align: middle;
        height: 60px;
        width: 60px;
    }
}

footer {
    text-align: center;
    font-size: 14px;
}

.main {
    transition: all ease-in-out 0.1s;
}

.maintenance-bg-info {
    color: $maintenance;
}

.maintenance-icon {
    font-size: 35px;
    vertical-align: middle;
}

.dark .shadow-box {
    background-color: #0d1117;
}

.status-maintenance {
    color: $maintenance;
    margin-right: 5px;
}

.mobile {
    h1 {
        font-size: 22px;
    }

    .overall-status {
        font-size: 20px;
    }
}

.bg-maintenance {
    .alert-heading {
        font-weight: bold;
    }
}

.refresh-info {
    opacity: 0.7;
}

.flow-chart-template {
    height: 500px;
}

</style>

<style lang="scss">
.mermaid {
    text-align: center;
}

.flowchart-link {
    stroke-dasharray: 100;
    /* animation: dashdraw 30s linear infinite reverse !important; */
}



.node.online > rect {
    fill: #4caf5054 !important;
    /* stroke: #333 !important; */
    /* stroke-width: 2px !important; */
    /* animation: pulse 1s infinite; */
    /* transition: stroke 1s ease-in-out; */

    /* animation: dashdraw 30s linear infinite reverse !important; */
}

.node.offline > rect {
    fill: #ff000045 !important;
    /*
    stroke: #333 !important;
    stroke-width: 2px !important;
    */

    animation: pulse-offline 2s infinite;
}

/*
@keyframes dashdraw {
    to {
        stroke-dashoffset: 200;
    }
}
*/


@keyframes pulse-offline {
  0% {
    stroke-width: 1px;
    stroke: gray;
  }
  70% {
    stroke-width: 3px;
    stroke: red;
  }
  100% {
    stroke-width: 1px;
    stroke: gray;
  }
}


</style>