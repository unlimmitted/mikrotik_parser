<template>
	<div class="app">
		<AppTopbar
			v-model:activeTab="activeTab"
			:ignore-v-p-n-label="ignoreVPNLabel"
			:ignore-lan-to-v-p-n-label="ignoreLanToVPNLabel"
			@update:activeTab="onTabChange"
		/>

		<main class="content">
			<IgnoreVpnTab
				v-if="activeTab === 'ignoreVPN'"
				:label="ignoreVPNLabel"
				:initial-filter="initialFilter"
				:initial-search-type="initialSearchType"
				@sync-url="onSyncUrl"
			/>
			<IgnoreLanToVpnTab
				v-else
				:label="ignoreLanToVPNLabel"
			/>
		</main>
	</div>
</template>

<script>
import axios from 'axios'
import AppTopbar from './components/AppTopbar.vue'
import IgnoreVpnTab from './components/IgnoreVpnTab.vue'
import IgnoreLanToVpnTab from './components/IgnoreLanToVpnTab.vue'

export default {
	name: 'App',
	components: { AppTopbar, IgnoreVpnTab, IgnoreLanToVpnTab },

	data() {
		return {
			activeTab: 'ignoreVPN',
			initialFilter: '',
			initialSearchType: 'byDomain',
			appConfig: {
				ignoreVPNListName: 'ignoreVpn',
				ignoreLanToVpnListName: 'ignoreLanToVpn',
			},
		}
	},

	computed: {
		ignoreVPNLabel() {
			return this.appConfig.ignoreVPNListName || 'ignoreVpn'
		},
		ignoreLanToVPNLabel() {
			return this.appConfig.ignoreLanToVpnListName || 'ignoreLanToVpn'
		},
	},

	methods: {
		async fetchAppConfig() {
			try {
				const { data } = await axios.get('/api/v1/config')
				this.appConfig = {
					ignoreVPNListName: String(data?.ignoreVPNListName || 'ignoreVpn').trim(),
					ignoreLanToVpnListName: String(data?.ignoreLanToVpnListName || 'ignoreLanToVpn').trim(),
				}
			} catch (e) {
				// оставляем дефолтные названия, чтобы UI не ломался при недоступности API
				console.log(e)
			}
		},

		onTabChange(tab) {
			this.activeTab = tab
			this.updateQueryParam('tab', tab)
		},

		onSyncUrl({ filter, searchType } = {}) {
			if (filter !== undefined) this.updateQueryParam('filter', filter)
			if (searchType !== undefined) this.updateQueryParam('searchType', searchType)
		},

		updateQueryParam(key, value) {
			const url = new URL(window.location)
			if (value === null || value === undefined || value === '') url.searchParams.delete(key)
			else url.searchParams.set(key, value)
			window.history.replaceState({}, '', url)
		},

		restoreFromUrl() {
			const params = new URLSearchParams(window.location.search)

			const tab = params.get('tab')
			if (tab === 'ignoreVPN' || tab === 'ignoreLanToVPN') this.activeTab = tab

			const st = params.get('searchType')
			if (st === 'byDomain' || st === 'bySrcAddress') this.initialSearchType = st

			const filter = params.get('filter')
			if (filter) this.initialFilter = filter
		},
	},

	async created() {
		await this.fetchAppConfig()
		this.restoreFromUrl()
	},
}
</script>

<style>
* { box-sizing: border-box; margin: 0; padding: 0; }
</style>

<style scoped>
.app {
	min-height: 100vh;
	background:
		radial-gradient(1200px 600px at 20% -10%, rgba(90, 66, 255, 0.15), transparent 60%),
		radial-gradient(900px 500px at 90% 10%, rgba(107, 87, 255, 0.12), transparent 55%),
		#f6f4ff;
	color: #1b1630;
	font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Arial, "Noto Sans", "Helvetica Neue", sans-serif;
}

.content {
	max-width: 1100px;
	margin: 0 auto;
	padding: 18px 20px 28px;
}
</style>
