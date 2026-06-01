<template>
	<div class="app">
		<header class="topbar">
			<div class="brand">
				<div class="logo">M</div>
				<div class="titles">
					<div class="title">MikroTik Parser</div>
					<div class="subtitle">Управление исключениями</div>
				</div>
			</div>

			<nav class="tabs" role="tablist" aria-label="tabs">
				<button
					class="tab"
					:class="{ active: activeTab === 'ignoreVPN' }"
					role="tab"
					:aria-selected="activeTab === 'ignoreVPN'"
					type="button"
					@click="setTab('ignoreVPN')"
				>
					{{ ignoreVPNLabel }}
				</button>
				<button
					class="tab"
					:class="{ active: activeTab === 'ignoreLanToVPN' }"
					role="tab"
					:aria-selected="activeTab === 'ignoreLanToVPN'"
					type="button"
					@click="setTab('ignoreLanToVPN')"
				>
					{{ ignoreLanToVPNLabel }}
				</button>
			</nav>
		</header>

		<main class="content">
			<!-- TAB 1: ignoreVPN -->
			<section v-if="activeTab === 'ignoreVPN'" class="card">
				<div class="card-header">
					<div>
						<div class="card-title">{{ ignoreVPNLabel }}</div>
						<div class="card-desc">Поиск доменов/источников и включение/выключение {{ ignoreVPNLabel }}</div>
					</div>

					<div class="status">
						<span v-if="loading" class="pill">Загрузка…</span>
						<span v-else class="pill muted">{{ domains.length }} записей</span>
					</div>
				</div>

				<div class="controls">
					<div class="field">
						<label class="label">Тип поиска</label>
						<div class="segmented">
							<button
								class="seg"
								:class="{ active: pickedSearchType === 'byDomain' }"
								type="button"
								@click="pickedSearchType = 'byDomain'; syncUrl()"
							>
								по домену
							</button>
							<button
								class="seg"
								:class="{ active: pickedSearchType === 'bySrcAddress' }"
								type="button"
								@click="pickedSearchType = 'bySrcAddress'; syncUrl()"
							>
								по src IP
							</button>
						</div>
					</div>

					<div class="field grow">
						<label class="label">
							<span v-if="pickedSearchType === 'byDomain'">DNS filter</span>
							<span v-else>Src address filter</span>
						</label>
						<input
							class="input"
							type="text"
							v-model="filterStr"
							ref="input"
							placeholder="Например: example.com или 192.168.1.10"
							@keydown.enter.prevent="filter"
						/>
					</div>

					<div class="field">
						<label class="label">Фильтр</label>
						<div class="segmented">
							<button
								class="seg"
								:class="{ active: vpnFilter === 'all' }"
								type="button"
								@click="vpnFilter = 'all'"
							>
								все
							</button>
							<button
								class="seg"
								:class="{ active: vpnFilter === 'enabled' }"
								type="button"
								@click="vpnFilter = 'enabled'"
							>
								включенные
							</button>
							<button
								class="seg"
								:class="{ active: vpnFilter === 'disabled' }"
								type="button"
								@click="vpnFilter = 'disabled'"
							>
								отключенные
							</button>
						</div>
					</div>

					<button class="btn primary" @click="filter" :disabled="!canSearch">
						Найти
					</button>
				</div>

				<div v-if="error" class="alert">
					<div class="alert-title">Ошибка</div>
					<div class="alert-text">{{ error }}</div>
				</div>

				<div class="list">
					<div v-for="domain in filteredDomains" :key="domain.id ?? domain.dstDns" class="row">
						<div class="row-main">
							<div class="dns">{{ domain.dstDns }}</div>

							<div class="chips" v-if="domain.dnsConnections && domain.dnsConnections.length">
                <span
	                class="chip"
	                v-for="(element, idx) in domain.dnsConnections"
	                :key="`${domain.dstDns}-${element.dstIP}-${idx}`"
	                :title="element.dstIP"
                >
                  {{ element.dstIP }}
                </span>
							</div>
							<div v-else class="muted small">Нет IP-адресов</div>
						</div>

						<div class="row-actions">
							<div class="toggle">
								<input
									class="toggle-input"
									type="checkbox"
									:id="`vpn-${domain.id ?? domain.dstDns}`"
									v-model="domain.isIgnoreVpn"
									@change="onIgnoreVpnToggle(domain)"
								/>
								<label class="toggle-label" :for="`vpn-${domain.id ?? domain.dstDns}`">
									<span class="toggle-track"></span>
									<span class="toggle-text">{{ ignoreVPNLabel }}</span>
								</label>
							</div>
						</div>
					</div>

					<div v-if="!loading && domains.length === 0" class="empty">
						Ничего не найдено
					</div>

					<div v-else-if="!loading && filteredDomains.length === 0" class="empty">
						По выбранному фильтру ничего нет
					</div>
				</div>
			</section>

			<!-- TAB 2: ignoreLanToVPN -->
			<section v-else class="card">
				<div class="card-header">
					<div>
						<div class="card-title">{{ ignoreLanToVPNLabel }}</div>
						<div class="card-desc">
							Ввод внутреннего адреса/подсети (IP или CIDR), которые нужно добавить в {{ ignoreLanToVPNLabel }}
						</div>
					</div>

					<div class="status">
						<span v-if="loadingLan" class="pill">Загрузка…</span>
						<span v-else class="pill muted">{{ lanIgnoreList.length }} адресов</span>
					</div>
				</div>

				<div class="controls">
					<div class="field grow">
						<label class="label">Внутренний адрес / подсеть</label>
						<input
							class="input"
							type="text"
							v-model="lanIgnoreInput"
							placeholder="Например: 192.168.0.10 или 10.0.0.0/24"
							@keydown.enter.prevent="addLanIgnore()"
						/>
					</div>

					<div class="field">
						<label class="label">Фильтр</label>
						<div class="segmented">
							<button class="seg" :class="{ active: lanFilter === 'all' }" type="button"
							        @click="lanFilter = 'all'">
								все
							</button>
							<button class="seg" :class="{ active: lanFilter === 'enabled' }" type="button"
							        @click="lanFilter = 'enabled'">
								включенные
							</button>
							<button class="seg" :class="{ active: lanFilter === 'disabled' }" type="button"
							        @click="lanFilter = 'disabled'">
								отключенные
							</button>
						</div>
					</div>

					<button class="btn primary" @click="addLanIgnore()" :disabled="!canAddLanIgnore">
						Добавить
					</button>
				</div>

				<div v-if="lanError" class="alert">
					<div class="alert-title">Ошибка</div>
					<div class="alert-text">{{ lanError }}</div>
				</div>

				<div class="list">
					<div v-for="item in filteredLanIgnoreList" :key="item.ip" class="row">
						<div class="row-main">
							<div class="dns">{{ item.hostName || '—' }}</div>
							<div class="muted small">{{ item.ip }}</div>
						</div>

						<div class="row-actions">
							<div class="toggle">
								<input
									class="toggle-input"
									type="checkbox"
									:id="`lan-${item.ip}`"
									v-model="item.enabled"
									@change="toggleLanIgnore(item)"
								/>
								<label class="toggle-label" :for="`lan-${item.ip}`">
									<span class="toggle-track"></span>
									<span class="toggle-text">{{ ignoreLanToVPNLabel }}</span>
								</label>
							</div>
						</div>
					</div>

					<div v-if="!loadingLan && lanIgnoreList.length === 0" class="empty">
						Пока нет адресов. Добавьте IP или подсеть.
					</div>

					<div v-else-if="!loadingLan && filteredLanIgnoreList.length === 0" class="empty">
						По выбранному фильтру ничего нет
					</div>
				</div>
			</section>
		</main>
	</div>
</template>

<script>
import axios from 'axios'

export default {
	data: () => ({
		// tabs
		activeTab: 'ignoreVPN',

		// config from backend env
		appConfig: {
			ignoreVPNListName: 'ignoreVpn',
			ignoreLanToVpnListName: 'ignoreLanToVpn',
		},

		// tab1: ignoreVPN
		domains: [],
		filterStr: '',
		pickedSearchType: 'byDomain',
		loading: false,
		error: '',

		// tab2: ignoreLanToVPN
		lanIgnoreInput: '',
		lanIgnoreList: [],
		lanError: '',
		lanFilter: 'all',
		loadingLan: false,
		vpnFilter: 'all',
		lanFindStr: '',
	}),

	computed: {
		ignoreVPNLabel() {
			return this.appConfig.ignoreVPNListName || 'ignoreVpn'
		},

		ignoreLanToVPNLabel() {
			return this.appConfig.ignoreLanToVpnListName || 'ignoreLanToVpn'
		},

		canSearch() {
			return !!(this.filterStr && this.filterStr.trim().length > 0 && this.pickedSearchType)
		},

		canAddLanIgnore() {
			return !!(this.lanIgnoreInput && this.lanIgnoreInput.trim().length > 0)
		},

		filteredLanIgnoreList() {
			const list = Array.isArray(this.lanIgnoreList) ? this.lanIgnoreList : []

			if (this.lanFilter === 'enabled') return list.filter(x => x.enabled === true)
			if (this.lanFilter === 'disabled') return list.filter(x => x.enabled === false)
			return list
		},

		filteredDomains() {
			const list = Array.isArray(this.domains) ? this.domains : []
			// isIgnoreVpn === true  -> "включено ignore vpn"
			// поэтому фильтр "включенные" = isIgnoreVpn true
			if (this.vpnFilter === 'enabled') return list.filter(x => x.isIgnoreVpn === true)
			if (this.vpnFilter === 'disabled') return list.filter(x => x.isIgnoreVpn === false)
			return list
		},
	},

	methods: {
		async fetchAppConfig() {
			try {
				const resp = await axios.get('/api/v1/config')
				this.appConfig = {
					ignoreVPNListName: String(resp.data?.ignoreVPNListName || 'ignoreVpn').trim(),
					ignoreLanToVpnListName: String(resp.data?.ignoreLanToVpnListName || 'ignoreLanToVpn').trim(),
				}
			} catch (e) {
				// оставляем дефолтные названия, чтобы UI не ломался при недоступности API
				console.log(e)
			}
		},

		async setTab(tab) {
			this.activeTab = tab
			this.syncUrl()

			if (tab === 'ignoreLanToVPN') {
				await this.fetchLanIgnoreList()
			}
		},

		// ---------- TAB 1 ----------
		async filter() {
			if (!this.canSearch) return

			this.error = ''
			this.loading = true

			const filter = (this.filterStr || '').trim()
			this.updateQueryParam('filter', filter)
			this.updateQueryParam('searchType', this.pickedSearchType)

			try {
				if (this.pickedSearchType === 'byDomain') {
					const response = await axios.get(`/api/v1/dns?find=${encodeURIComponent(filter)}`)
					this.domains = Array.isArray(response.data) ? response.data : []
				} else {
					const response = await axios.get(`/api/v1/src?srcIp=${encodeURIComponent(filter)}`)
					this.domains = Array.isArray(response.data) ? response.data : []
				}
			} catch (e) {
				this.error = 'Не удалось выполнить запрос. Проверьте доступность API и параметры.'
				// eslint-disable-next-line no-console
				console.log(e)
			} finally {
				this.loading = false
			}
		},

		async onIgnoreVpnToggle(domain) {
			// Текущее поведение: enabled = НЕ ignoreVpn
			// Если на бэке наоборот — поменяйте здесь.
			this.error = ''
			try {
				await axios.post(
					`/api/v1/dns?dns=${encodeURIComponent(domain.dstDns)}&enabled=${encodeURIComponent(domain.isIgnoreVpn)}`
				)
			} catch (e) {
				// откат UI при ошибке
				domain.isIgnoreVpn = !domain.isIgnoreVpn
				this.error = `Не удалось обновить ${this.ignoreVPNLabel} для домена.`
				// eslint-disable-next-line no-console
				console.log(e)
			}
		},

		// ---------- TAB 2 ----------
		async fetchLanIgnoreList() {
			this.lanError = ''
			this.loadingLan = true
			try {
				const resp = await axios.get('/api/v1/ignore-lan-to-vpn', {
					params: { find: (this.lanFindStr || '').trim() } // используем ваш инпут как фильтр
				})

				this.lanIgnoreList = Array.isArray(resp.data)
					? resp.data
						.map(x => ({
							ip: String(x.ip ?? '').trim(),
							hostName: String(x.hostName ?? '').trim(),
							enabled: x.enabled === true,
							dynamic: x.dynamic === true,
						}))
						.filter(x => x.ip.length > 0)
					: []

			} catch (e) {
				this.lanError = `Не удалось загрузить список ${this.ignoreLanToVPNLabel}.`
				// eslint-disable-next-line no-console
				console.log(e)
			} finally {
				this.loadingLan = false
			}
		},

		async addLanIgnore() {
			this.lanError = ''
			const ip = (this.lanIgnoreInput || '').trim()
			if (!ip) return

			if (!this.isIpOrCidr(ip)) {
				this.lanError = 'Неверный формат. Введите IP (x.x.x.x) или CIDR (x.x.x.x/nn).'
				return
			}

			try {
				await axios.post('/api/v1/ignore-lan-to-vpn', {
					ip,
					enabled: true,
					hostName: '' // будет подтянуто с бэка по DHCP leases при следующем fetch
				})
				this.lanIgnoreInput = ''
				await this.fetchLanIgnoreList()
			} catch (e) {
				this.lanError = `Не удалось добавить адрес в ${this.ignoreLanToVPNLabel}.`
				console.log(e)
			}
		},

		async toggleLanIgnore(item) {
			this.lanError = ''
			const prev = !item.enabled
			try {
				await axios.post('/api/v1/ignore-lan-to-vpn', {
					ip: item.ip,
					enabled: item.enabled,
					hostName: item.hostName,
				})
			} catch (e) {
				item.enabled = prev
				this.lanError = `Не удалось обновить ${this.ignoreLanToVPNLabel}.`
				console.log(e)
			}
		},

		isIpOrCidr(value) {
			// базовая проверка IPv4 и IPv4/CIDR
			const ip = '(?:25[0-5]|2[0-4]\\d|1\\d\\d|[1-9]?\\d)'
			const ipRegex = new RegExp(`^${ip}\\.${ip}\\.${ip}\\.${ip}$`)
			const cidrRegex = new RegExp(`^${ip}\\.${ip}\\.${ip}\\.${ip}\\/(?:[0-9]|[1-2]\\d|3[0-2])$`)
			return ipRegex.test(value) || cidrRegex.test(value)
		},

		// ---------- URL helpers ----------
		updateQueryParam(key, value) {
			const url = new URL(window.location)
			if (value === null || value === undefined || value === '') url.searchParams.delete(key)
			else url.searchParams.set(key, value)
			window.history.replaceState({}, '', url)
		},

		syncUrl() {
			this.updateQueryParam('tab', this.activeTab)
			this.updateQueryParam('searchType', this.pickedSearchType)
			this.updateQueryParam('filter', (this.filterStr || '').trim())
		},

		restoreFromUrl() {
			const params = new URLSearchParams(window.location.search)

			const tab = params.get('tab')
			if (tab === 'ignoreVPN' || tab === 'ignoreLanToVPN') this.activeTab = tab

			const st = params.get('searchType')
			if (st === 'byDomain' || st === 'bySrcAddress') this.pickedSearchType = st

			const filter = params.get('filter')
			if (filter) this.filterStr = filter
		},
	},

	async mounted() {
		await this.fetchAppConfig()
		this.restoreFromUrl()

		// список вкладки 2 грузим сразу (можно грузить лениво при переключении)
		await this.fetchLanIgnoreList()

		if (this.activeTab === 'ignoreVPN' && this.filterStr) {
			await this.filter()
		}

		if (this.activeTab === 'ignoreLanToVPN') {
			await this.fetchLanIgnoreList()
		}
	},
}
</script>

<style scoped>
.app {
	min-height: 100vh;
	background: radial-gradient(1200px 600px at 20% -10%, rgba(90, 66, 255, 0.15), transparent 60%),
	radial-gradient(900px 500px at 90% 10%, rgba(107, 87, 255, 0.12), transparent 55%),
	#f6f4ff;
	color: #1b1630;
	font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Arial, "Noto Sans", "Helvetica Neue", sans-serif;
}

.topbar {
	position: sticky;
	top: 0;
	z-index: 10;
	display: flex;
	align-items: center;
	justify-content: space-between;
	gap: 16px;
	padding: 18px 20px;
	border-bottom: 1px solid rgba(20, 18, 40, 0.10);
	background: rgba(255, 255, 255, 0.70);
	backdrop-filter: blur(10px);
}

.brand {
	display: flex;
	align-items: center;
	gap: 12px;
	min-width: 240px;
}

.logo {
	width: 38px;
	height: 38px;
	border-radius: 12px;
	display: grid;
	place-items: center;
	font-weight: 800;
	color: white;
	background: linear-gradient(135deg, #5a42ff, #6b57ff);
	box-shadow: 0 8px 24px rgba(90, 66, 255, 0.25);
}

.titles .title {
	font-weight: 700;
	letter-spacing: 0.2px;
}

.titles .subtitle {
	font-size: 12px;
	opacity: 0.65;
	margin-top: 2px;
}

.tabs {
	display: flex;
	gap: 8px;
	padding: 6px;
	border-radius: 16px;
	background: rgba(20, 18, 40, 0.04);
	border: 1px solid rgba(20, 18, 40, 0.08);
}

.tab {
	border: 0;
	background: transparent;
	padding: 10px 14px;
	border-radius: 12px;
	cursor: pointer;
	font-weight: 700;
	color: rgba(27, 22, 48, 0.75);
	transition: transform .08s ease, background .15s ease, color .15s ease;
}

.tab:hover {
	background: rgba(90, 66, 255, 0.08);
}

.tab:active {
	transform: translateY(1px);
}

.tab.active {
	background: white;
	color: #1b1630;
	box-shadow: 0 8px 18px rgba(20, 18, 40, 0.10);
	border: 1px solid rgba(20, 18, 40, 0.08);
}

.content {
	max-width: 1100px;
	margin: 0 auto;
	padding: 18px 20px 28px;
}

.card {
	background: rgba(255, 255, 255, 0.78);
	border: 1px solid rgba(20, 18, 40, 0.10);
	border-radius: 18px;
	box-shadow: 0 18px 40px rgba(20, 18, 40, 0.10);
	overflow: hidden;
}

.card-header {
	display: flex;
	align-items: flex-start;
	justify-content: space-between;
	gap: 16px;
	padding: 18px 18px 14px;
	border-bottom: 1px solid rgba(20, 18, 40, 0.08);
}

.card-title {
	font-weight: 900;
	letter-spacing: 0.2px;
}

.card-desc {
	margin-top: 4px;
	font-size: 13px;
	opacity: 0.70;
	line-height: 1.35;
	max-width: 720px;
}

.status {
	display: flex;
	align-items: center;
	gap: 10px;
}

.pill {
	font-size: 12px;
	font-weight: 800;
	padding: 8px 10px;
	border-radius: 999px;
	background: rgba(90, 66, 255, 0.10);
	color: #3b2ee0;
	border: 1px solid rgba(90, 66, 255, 0.18);
}

.pill.muted {
	background: rgba(20, 18, 40, 0.05);
	color: rgba(27, 22, 48, 0.70);
	border-color: rgba(20, 18, 40, 0.08);
}

.controls {
	display: flex;
	align-items: flex-end;
	gap: 12px;
	padding: 14px 18px 18px;
	border-bottom: 1px solid rgba(20, 18, 40, 0.06);
}

.field {
	display: flex;
	flex-direction: column;
	gap: 6px;
	min-width: 220px;
}

.field.grow {
	flex: 1;
	min-width: 260px;
}

.label {
	font-size: 12px;
	font-weight: 800;
	opacity: 0.75;
}

.input {
	height: 42px;
	border-radius: 12px;
	border: 1px solid rgba(20, 18, 40, 0.10);
	background: rgba(255, 255, 255, 0.85);
	padding: 0 12px;
	outline: none;
	transition: border-color .15s ease, box-shadow .15s ease, background .15s ease;
}

.input:focus {
	border-color: rgba(90, 66, 255, 0.45);
	box-shadow: 0 0 0 4px rgba(90, 66, 255, 0.10);
	background: #fff;
}

.hint {
	font-size: 12px;
	opacity: 0.6;
}

.btn {
	height: 42px;
	border-radius: 12px;
	padding: 0 14px;
	font-weight: 900;
	border: 1px solid rgba(20, 18, 40, 0.10);
	background: rgba(255, 255, 255, 0.85);
	cursor: pointer;
	transition: transform .08s ease, box-shadow .15s ease, background .15s ease;
}

.btn:active {
	transform: translateY(1px);
}

.btn:disabled {
	opacity: 0.5;
	cursor: not-allowed;
}

.btn.primary {
	border-color: rgba(90, 66, 255, 0.35);
	background: linear-gradient(135deg, rgba(90, 66, 255, 0.92), rgba(107, 87, 255, 0.92));
	color: #fff;
	box-shadow: 0 12px 28px rgba(90, 66, 255, 0.25);
}

.btn.primary:hover {
	box-shadow: 0 14px 30px rgba(90, 66, 255, 0.30);
}

.btn.ghost {
	background: rgba(20, 18, 40, 0.04);
	border-color: rgba(20, 18, 40, 0.08);
}

.segmented {
	display: flex;
	gap: 6px;
	padding: 6px;
	border-radius: 14px;
	background: rgba(20, 18, 40, 0.04);
	border: 1px solid rgba(20, 18, 40, 0.08);
}

.seg {
	border: 0;
	background: transparent;
	padding: 8px 10px;
	border-radius: 12px;
	cursor: pointer;
	font-weight: 900;
	color: rgba(27, 22, 48, 0.70);
	transition: background .15s ease, color .15s ease;
}

.seg:hover {
	background: rgba(90, 66, 255, 0.08);
}

.seg.active {
	background: #fff;
	color: #1b1630;
	border: 1px solid rgba(20, 18, 40, 0.08);
	box-shadow: 0 10px 18px rgba(20, 18, 40, 0.08);
}

.alert {
	margin: 14px 18px 0;
	border-radius: 14px;
	border: 1px solid rgba(255, 0, 80, 0.18);
	background: rgba(255, 0, 80, 0.06);
	padding: 12px 12px;
}

.alert-title {
	font-weight: 900;
	margin-bottom: 4px;
}

.alert-text {
	font-size: 13px;
	opacity: 0.85;
	line-height: 1.35;
}

.list {
	padding: 10px 10px 12px;
}

.row {
	display: flex;
	align-items: center;
	justify-content: space-between;
	gap: 14px;
	padding: 12px 12px;
	margin: 10px 8px;
	border-radius: 16px;
	border: 1px solid rgba(20, 18, 40, 0.08);
	background: rgba(255, 255, 255, 0.86);
	box-shadow: 0 10px 22px rgba(20, 18, 40, 0.06);
}

.row-main {
	display: flex;
	flex-direction: column;
	gap: 6px;
	min-width: 0;
}

.dns {
	font-weight: 950;
	letter-spacing: 0.1px;
	word-break: break-word;
}

.chips {
	display: flex;
	flex-wrap: wrap;
	gap: 6px;
}

.chip {
	font-size: 12px;
	font-weight: 800;
	padding: 6px 10px;
	border-radius: 999px;
	border: 1px solid rgba(20, 18, 40, 0.08);
	background: rgba(20, 18, 40, 0.04);
	opacity: 0.95;
}

.row-actions {
	display: flex;
	align-items: center;
	gap: 10px;
	flex-shrink: 0;
}

.toggle {
	display: flex;
	align-items: center;
}

.toggle-input {
	position: absolute;
	opacity: 0;
	pointer-events: none;
}

.toggle-label {
	display: inline-flex;
	align-items: center;
	gap: 10px;
	cursor: pointer;
	user-select: none;
}

.toggle-track {
	width: 46px;
	height: 28px;
	border-radius: 999px;
	border: 1px solid rgba(20, 18, 40, 0.12);
	background: rgba(20, 18, 40, 0.08);
	position: relative;
	transition: background .15s ease, border-color .15s ease;
}

.toggle-track::after {
	content: "";
	width: 22px;
	height: 22px;
	border-radius: 999px;
	background: #fff;
	position: absolute;
	top: 50%;
	left: 3px;
	transform: translateY(-50%);
	box-shadow: 0 8px 16px rgba(20, 18, 40, 0.18);
	transition: left .15s ease;
}

.toggle-text {
	font-size: 12px;
	font-weight: 900;
	opacity: 0.75;
}

.toggle-input:checked + .toggle-label .toggle-track {
	background: rgba(90, 66, 255, 0.25);
	border-color: rgba(90, 66, 255, 0.35);
}

.toggle-input:checked + .toggle-label .toggle-track::after {
	left: 21px;
}

.muted {
	opacity: 0.65;
}

.small {
	font-size: 12px;
}

.empty {
	padding: 18px 18px 22px;
	text-align: center;
	opacity: 0.7;
	font-weight: 800;
}

.note {
	margin: 10px 18px 18px;
	border-radius: 16px;
	border: 1px solid rgba(20, 18, 40, 0.08);
	background: rgba(20, 18, 40, 0.03);
	padding: 12px 12px;
}

.note-title {
	font-weight: 950;
	margin-bottom: 6px;
}

.note-text {
	font-size: 13px;
	opacity: 0.85;
	line-height: 1.35;
}

code {
	font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace;
	font-size: 12px;
	padding: 2px 6px;
	border-radius: 8px;
	background: rgba(255, 255, 255, 0.8);
	border: 1px solid rgba(20, 18, 40, 0.08);
}

@media (max-width: 860px) {
	.topbar {
		flex-direction: column;
		align-items: stretch;
	}

	.brand {
		min-width: 0;
	}

	.controls {
		flex-direction: column;
		align-items: stretch;
	}

	.field {
		min-width: 0;
	}

	.row {
		flex-direction: column;
		align-items: stretch;
	}

	.row-actions {
		justify-content: space-between;
	}
}
</style>
