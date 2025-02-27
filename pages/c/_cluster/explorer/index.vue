<script>
import Loading from '@/components/Loading';
import DashboardMetrics from '@/components/DashboardMetrics';
import { mapGetters } from 'vuex';
import SortableTable from '@/components/SortableTable';
import { allHash } from '@/utils/promise';
import AlertTable from '@/components/AlertTable';
import Banner from '@/components/Banner';
import { parseSi, createMemoryValues } from '@/utils/units';
import {
  NAME,
  REASON,
  ROLES,
  STATE,
} from '@/config/table-headers';
import {
  NAMESPACE,
  INGRESS,
  EVENT,
  MANAGEMENT,
  METRIC,
  NODE,
  SERVICE,
  PV,
  WORKLOAD_TYPES,
  COUNT,
  CATALOG,
} from '@/config/types';
import { findBy } from '@/utils/array';
import { mapPref, CLUSTER_TOOLS_TIP } from '@/store/prefs';
import { haveV1Monitoring, monitoringStatus } from '@/utils/monitoring';
import Tabbed from '@/components/Tabbed';
import Tab from '@/components/Tabbed/Tab';
import { allDashboardsExist } from '@/utils/grafana';
import EtcdInfoBanner from '@/components/EtcdInfoBanner';
import metricPoller from '@/mixins/metric-poller';
import EmberPage from '@/components/EmberPage';
import ResourceSummary, { resourceCounts } from '@/components/ResourceSummary';
import HardwareResourceGauge from '@/components/HardwareResourceGauge';

export const RESOURCES = [NAMESPACE, INGRESS, PV, WORKLOAD_TYPES.DEPLOYMENT, WORKLOAD_TYPES.STATEFUL_SET, WORKLOAD_TYPES.JOB, WORKLOAD_TYPES.DAEMON_SET, SERVICE];

const CLUSTER_METRICS_DETAIL_URL = '/api/v1/namespaces/cattle-monitoring-system/services/http:rancher-monitoring-grafana:80/proxy/d/rancher-cluster-nodes-1/rancher-cluster-nodes?orgId=1';
const CLUSTER_METRICS_SUMMARY_URL = '/api/v1/namespaces/cattle-monitoring-system/services/http:rancher-monitoring-grafana:80/proxy/d/rancher-cluster-1/rancher-cluster?orgId=1';
const K8S_METRICS_DETAIL_URL = '/api/v1/namespaces/cattle-monitoring-system/services/http:rancher-monitoring-grafana:80/proxy/d/rancher-k8s-components-nodes-1/rancher-kubernetes-components-nodes?orgId=1';
const K8S_METRICS_SUMMARY_URL = '/api/v1/namespaces/cattle-monitoring-system/services/http:rancher-monitoring-grafana:80/proxy/d/rancher-k8s-components-1/rancher-kubernetes-components?orgId=1';
const ETCD_METRICS_DETAIL_URL = '/api/v1/namespaces/cattle-monitoring-system/services/http:rancher-monitoring-grafana:80/proxy/d/rancher-etcd-nodes-1/rancher-etcd-nodes?orgId=1';
const ETCD_METRICS_SUMMARY_URL = '/api/v1/namespaces/cattle-monitoring-system/services/http:rancher-monitoring-grafana:80/proxy/d/rancher-etcd-1/rancher-etcd?orgId=1';

export default {
  components: {
    EtcdInfoBanner,
    DashboardMetrics,
    HardwareResourceGauge,
    Loading,
    ResourceSummary,
    SortableTable,
    Tab,
    Tabbed,
    AlertTable,
    Banner,
    EmberPage,
  },

  mixins: [metricPoller],

  async fetch() {
    const hash = {
      nodes:       this.fetchClusterResources(NODE),
      events:      this.fetchClusterResources(EVENT),
    };

    if ( this.$store.getters['management/canList'](MANAGEMENT.NODE_TEMPLATE) ) {
      hash.nodeTemplates = this.$store.dispatch('management/findAll', { type: MANAGEMENT.NODE_TEMPLATE });
    }

    if ( this.$store.getters['management/canList'](MANAGEMENT.NODE_POOL) ) {
      hash.rke1NodePools = this.$store.dispatch('management/findAll', { type: MANAGEMENT.NODE_POOL });
    }

    this.showClusterMetrics = await allDashboardsExist(this.$store.dispatch, this.currentCluster.id, [CLUSTER_METRICS_DETAIL_URL, CLUSTER_METRICS_SUMMARY_URL]);
    this.showK8sMetrics = await allDashboardsExist(this.$store.dispatch, this.currentCluster.id, [K8S_METRICS_DETAIL_URL, K8S_METRICS_SUMMARY_URL]);
    this.showEtcdMetrics = this.isRKE && await allDashboardsExist(this.$store.dispatch, this.currentCluster.id, [ETCD_METRICS_DETAIL_URL, ETCD_METRICS_SUMMARY_URL]);

    const res = await allHash(hash);

    for ( const k in res ) {
      this[k] = res[k];
    }
    this.metricAggregations = await this.currentCluster.fetchNodeMetrics();
  },

  data() {
    const clusterCounts = this.$store.getters[`cluster/all`](COUNT);
    const reason = {
      ...REASON,
      ...{ canBeVariable: true },
      width: 120
    };

    const eventHeaders = [
      reason,
      {
        name:          'resource',
        label:         'Resource',
        labelKey:      'clusterIndexPage.sections.events.resource.label',
        value:         'displayInvolvedObject',
        sort:          ['involvedObject.kind', 'involvedObject.name'],
        canBeVariable: true,
      },
      {
        align:         'right',
        name:          'date',
        label:         'Date',
        labelKey:      'clusterIndexPage.sections.events.date.label',
        value:         'lastTimestamp',
        sort:          'lastTimestamp:desc',
        formatter:     'LiveDate',
        formatterOpts: { addSuffix: true },
        width:         125,
        defaultSort:   true,
      },
    ];

    const nodeHeaders = [
      STATE,
      NAME,
      ROLES,
    ];

    return {
      eventHeaders,
      nodeHeaders,
      constraints:         [],
      events:              [],
      nodeMetrics:         [],
      nodeTemplates:       [],
      nodes:               [],
      metricAggregations: {},
      showClusterMetrics: false,
      showK8sMetrics:     false,
      showEtcdMetrics:    false,
      CLUSTER_METRICS_DETAIL_URL,
      CLUSTER_METRICS_SUMMARY_URL,
      K8S_METRICS_DETAIL_URL,
      K8S_METRICS_SUMMARY_URL,
      ETCD_METRICS_DETAIL_URL,
      ETCD_METRICS_SUMMARY_URL,
      clusterCounts
    };
  },

  computed: {
    ...mapGetters(['currentCluster']),
    ...monitoringStatus(),

    hideClusterToolsTip: mapPref(CLUSTER_TOOLS_TIP),

    hasV1Monitoring() {
      return haveV1Monitoring(this.$store.getters);
    },

    v1MonitoringURL() {
      return `/k/${ this.currentCluster.id }/monitoring`;
    },

    displayProvider() {
      const other = 'other';

      let provider = this.currentCluster.status.provider || other;

      if (provider === 'rke.windows') {
        provider = 'rkeWindows';
      }

      if (!this.$store.getters['i18n/exists'](`cluster.provider.${ provider }`)) {
        provider = 'other';
      }

      return this.t(`cluster.provider.${ provider }`);
    },

    isRKE() {
      return ['rke', 'rke.windows', 'rke2', 'rke2.windows'].includes((this.currentCluster.status.provider || '').toLowerCase());
    },

    accessibleResources() {
      return RESOURCES.filter(resource => this.$store.getters['cluster/schemaFor'](resource));
    },

    totalCountGaugeInput() {
      const totalInput = {
        name:            this.t('clusterIndexPage.resourceGauge.totalResources'),
        total:           0,
        useful:          0,
        warningCount:    0,
        errorCount:      0
      };

      this.accessibleResources.forEach((resource) => {
        const counts = resourceCounts(this.$store, resource);

        Object.entries(counts).forEach((entry) => {
          totalInput[entry[0]] += entry[1];
        });
      });

      return totalInput;
    },

    hasStats() {
      return this.currentCluster?.status?.allocatable && this.currentCluster?.status?.requested;
    },

    cpuReserved() {
      return {
        total:  parseSi(this.currentCluster?.status?.allocatable?.cpu),
        useful: parseSi(this.currentCluster?.status?.requested?.cpu)
      };
    },

    podsUsed() {
      return {
        total:  parseSi(this.currentCluster?.status?.allocatable?.pods || '0'),
        useful: parseSi(this.currentCluster?.status?.requested?.pods || '0')
      };
    },

    ramReserved() {
      return createMemoryValues(this.currentCluster?.status?.allocatable?.memory, this.currentCluster?.status?.requested?.memory);
    },

    cpuUsed() {
      return {
        total:  parseSi(this.currentCluster?.status?.capacity?.cpu),
        useful: this.metricAggregations?.cpu
      };
    },

    ramUsed() {
      return createMemoryValues(this.currentCluster?.status?.capacity?.memory, this.metricAggregations?.memory);
    },

    hasMonitoring() {
      return !!this.clusterCounts?.[0]?.counts?.[CATALOG.APP]?.namespaces?.['cattle-monitoring-system'];
    },

    canAccessNodes() {
      return !!this.clusterCounts?.[0]?.counts?.[NODE];
    },

    canAccessDeployments() {
      return !!this.clusterCounts?.[0]?.counts?.[WORKLOAD_TYPES.DEPLOYMENT];
    },

    hasMetricsTabs() {
      return this.showClusterMetrics || this.showK8sMetrics || this.showEtcdMetrics;
    }
  },

  methods: {
    showActions() {
      this.$store.commit('action-menu/show', {
        resources: this.currentCluster,
        elem:      this.$refs['cluster-actions'],
      });
    },

    async fetchClusterResources(type, opt = {}) {
      const schema = this.$store.getters['cluster/schemaFor'](type);

      if (schema) {
        try {
          const resources = await this.$store.dispatch('cluster/findAll', { type, opt });

          return resources;
        } catch (err) {
          console.error(`Failed fetching cluster resource ${ type } with error:`, err); // eslint-disable-line no-console

          return [];
        }
      }

      return [];
    },

    async loadMetrics() {
      this.nodeMetrics = await this.fetchClusterResources(METRIC.NODE, { force: true } );
    },
    findBy,
  },

};
</script>

<template>
  <Loading v-if="$fetchState.pending" />
  <section v-else class="dashboard">
    <header>
      <div class="title">
        <h1>
          <t k="clusterIndexPage.header" />
        </h1>
        <div>
          <span v-if="currentCluster.spec.description">{{ currentCluster.spec.description }}</span>
        </div>
      </div>
    </header>
    <Banner
      v-if="!hideClusterToolsTip"
      :closable="true"
      class="cluster-tools-tip"
      color="info"
      label-key="cluster.toolsTip"
      @close="hideClusterToolsTip = true"
    />
    <div
      class="cluster-dashboard-glance"
    >
      <div>
        <label>{{ t('glance.provider') }}: </label>
        <span>
          {{ displayProvider }}</span>
      </div>
      <div>
        <label>{{ t('glance.version') }}: </label>
        <span v-if="currentCluster.kubernetesVersionExtension" style="font-size: 0.5em">{{ currentCluster.kubernetesVersionExtension }}</span>
        <span>{{ currentCluster.kubernetesVersionBase }}</span>
      </div>
      <div>
        <label>{{ t('glance.created') }}: </label>
        <span><LiveDate :value="currentCluster.metadata.creationTimestamp" :add-suffix="true" :show-tooltip="true" /></span>
      </div>
      <div :style="{'flex':1}" />
      <div v-if="!monitoringStatus.v2 && !monitoringStatus.v1">
        <n-link :to="{name: 'c-cluster-explorer-tools'}" class="monitoring-install">
          <i class="icon icon-gear" />
          <span>{{ t('glance.installMonitoring') }}</span>
        </n-link>
      </div>
      <div v-if="monitoringStatus.v1">
        <span>{{ t('glance.v1MonitoringInstalled') }}</span>
      </div>
    </div>

    <div class="resource-gauges">
      <ResourceSummary :spoofed-counts="totalCountGaugeInput" />
      <ResourceSummary v-if="canAccessNodes" resource="node" />
      <ResourceSummary v-if="canAccessDeployments" resource="apps.deployment" />
    </div>

    <h3 v-if="!hasV1Monitoring && hasStats" class="mt-40">
      {{ t('clusterIndexPage.sections.capacity.label') }}
    </h3>
    <div v-if="!hasV1Monitoring && hasStats" class="hardware-resource-gauges">
      <HardwareResourceGauge :name="t('clusterIndexPage.hardwareResourceGauge.pods')" :used="podsUsed" />
      <HardwareResourceGauge :name="t('clusterIndexPage.hardwareResourceGauge.cores')" :reserved="cpuReserved" :used="cpuUsed" />
      <HardwareResourceGauge :name="t('clusterIndexPage.hardwareResourceGauge.ram')" :reserved="ramReserved" :used="ramUsed" :units="ramReserved.units" />
    </div>

    <div v-if="hasV1Monitoring" id="ember-anchor" class="mt-20">
      <EmberPage inline="ember-anchor" :src="v1MonitoringURL" />
    </div>

    <div class="mb-40 mt-40">
      <h3>{{ hasMonitoring ?t('clusterIndexPage.sections.alerts.label') :t('clusterIndexPage.sections.events.label') }}</h3>
      <AlertTable v-if="hasMonitoring" />
      <SortableTable
        v-else
        :rows="events"
        :headers="eventHeaders"
        key-field="id"
        :search="false"
        :table-actions="false"
        :row-actions="false"
        :paging="true"
        :rows-per-page="10"
        default-sort-by="date"
      >
        <template #cell:resource="{row, value}">
          <n-link :to="row.detailLocation">
            {{ value }}
          </n-link>
          <div v-if="row.message">
            {{ row.displayMessage }}
          </div>
        </template>
      </SortableTable>
    </div>
    <Tabbed v-if="hasMetricsTabs" class="mt-30">
      <Tab v-if="showClusterMetrics" name="cluster-metrics" :label="t('clusterIndexPage.sections.clusterMetrics.label')" :weight="2">
        <template #default="props">
          <DashboardMetrics
            v-if="props.active"
            :detail-url="CLUSTER_METRICS_DETAIL_URL"
            :summary-url="CLUSTER_METRICS_SUMMARY_URL"
            graph-height="825px"
          />
        </template>
      </Tab>
      <Tab v-if="showK8sMetrics" name="k8s-metrics" :label="t('clusterIndexPage.sections.k8sMetrics.label')" :weight="1">
        <template #default="props">
          <DashboardMetrics
            v-if="props.active"
            :detail-url="K8S_METRICS_DETAIL_URL"
            :summary-url="K8S_METRICS_SUMMARY_URL"
            graph-height="550px"
          />
        </template>
      </Tab>
      <Tab v-if="showEtcdMetrics" name="etcd-metrics" :label="t('clusterIndexPage.sections.etcdMetrics.label')" :weight="0">
        <template #default="props">
          <DashboardMetrics
            v-if="props.active"
            class="etcd-metrics"
            :detail-url="ETCD_METRICS_DETAIL_URL"
            :summary-url="ETCD_METRICS_SUMMARY_URL"
            graph-height="550px"
          >
            <EtcdInfoBanner />
          </DashboardMetrics>
        </template>
      </Tab>
    </Tabbed>
  </section>
</template>

<style lang="scss" scoped>
.cluster-dashboard-glance {
  border-top: 1px solid var(--border);
  border-bottom: 1px solid var(--border);
  padding: 20px 0px;
  display: flex;

  &>*:not(:last-child) {
    margin-right: 40px;

    & SPAN {
       font-weight: bold
    }
  }
}

.title h1 {
  margin: 0;
}

.actions-span {
  align-self: center;
}

.events {
  margin-top: 30px;
}

.graph-options {
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  align-items: center;
}

.etcd-metrics ::v-deep .external-link {
  top: -102px;
}

.cluster-tools-tip {
  margin-top: 0;
}

.monitoring-install {
  display: flex;

  > I {
    line-height: inherit;
    margin-right: 4px;
  }

  &:focus {
    outline: 0;
  }
}
</style>
