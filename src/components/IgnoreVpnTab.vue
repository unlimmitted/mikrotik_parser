<template>
	<section class="card">
		<div class="card-header">
			<div>
				<div class="card-title">{{ label }}</div>
				<div class="card-desc">Поиск доменов/источников и включение/выключение {{ label }}</div>
			</div>
			<div class="status">
				<span v-if="loading" class="pill">Загрузка…</span>
				<span v-else class="pill muted">{{ domains.length }} записей</span>
			</div>
		</div>

		<div class="controls">
			<div class="field">
				<label class="label">Тип поиска</label>
				<SegmentedControl
					v-model="pickedSearchType"
					:options="searchTypeOptions"
					@update:modelValue="onSearchTypeChange"
				/>
			</div>

			<div class="field grow">
				<label class="label">
					<span v-if="pickedSearchType === 'byDomain'">Фильтр по DNS</span>
					<span v-else>Фильтр по IP устройства</span>
				</label>
				<input
					class="input"
					type="text"
					v-model="filterStr"
					ref="input"
					placeholder="Например: example.com или 192.168.1.10"
					@keydown.enter.prevent="doFilter"
				/>
			</div>

			<div class="field">
				<label class="label">Фильтр</label>
				<SegmentedControl v-model="vpnFilter" :options="statusFilterOptions" />
			</div>

			<button class="btn primary" @click="doFilter" :disabled="!canSearch">
				Найти
			</button>
		</div>

		<AppAlert :message="error" />

		<div class="list">
			<div
          v-for="domain in filteredDomains"
          :key="domain.id ?? domain.dstDns"
          class="row"
      >
				<div class="row-main">
					<div
              v-if="domain.dstDns.length !== 0"
              class="dns"
          >
            {{ domain.dstDns }}
          </div>
					<div class="chips" >
						<span
							class="chip"
							v-for="(element, idx) in domain.dnsConnections"
							:key="`${domain.dstDns}-${element.dstIP}-${idx}`"
							:title="element.dstIP"
						>
							{{ element.dstIP }}
						</span>
					</div>
				</div>

				<div class="row-actions">
					<ToggleSwitch
						:input-id="`vpn-${domain.id ?? domain.dstDns}`"
						:model-value="domain.isIgnoreVpn"
						:label="label"
						@update:modelValue="val => onToggle(domain, val)"
					/>
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
</template>

<script>
import axios from 'axios'
import AppAlert from './AppAlert.vue'
import SegmentedControl from './SegmentedControl.vue'
import ToggleSwitch from './ToggleSwitch.vue'

export default {
	name: 'IgnoreVpnTab',
	components: { AppAlert, SegmentedControl, ToggleSwitch },

	props: {
		label: { type: String, default: 'ignoreVpn' },
		initialFilter: { type: String, default: '' },
		initialSearchType: { type: String, default: 'byDomain' },
	},

	emits: ['sync-url'],

	data() {
		return {
			domains: [],
			filterStr: this.initialFilter,
			pickedSearchType: this.initialSearchType,
			loading: false,
			error: '',
			vpnFilter: 'all',

			searchTypeOptions: [
				{ value: 'byDomain', label: 'по домену' },
				{ value: 'bySrcAddress', label: 'по входящему IP' },
			],
			statusFilterOptions: [
				{ value: 'all', label: 'все' },
				{ value: 'enabled', label: 'включенные' },
				{ value: 'disabled', label: 'отключенные' },
			],
		}
	},

	computed: {
		canSearch() {
			return !!(this.filterStr && this.filterStr.trim().length > 0 && this.pickedSearchType)
		},
		filteredDomains() {
			const list = Array.isArray(this.domains) ? this.domains : []
			if (this.vpnFilter === 'enabled') return list.filter(x => x.isIgnoreVpn === true)
			if (this.vpnFilter === 'disabled') return list.filter(x => x.isIgnoreVpn === false)
			return list.filter(d => d.dstDns?.length > 0)
		},
	},

	methods: {
		onSearchTypeChange() {
			this.$emit('sync-url', { filter: this.filterStr.trim(), searchType: this.pickedSearchType })
		},

		async doFilter() {
			if (!this.canSearch) return
			this.error = ''
			this.loading = true

			const f = (this.filterStr || '').trim()
			this.$emit('sync-url', { filter: f, searchType: this.pickedSearchType })

			try {
				if (this.pickedSearchType === 'byDomain') {
					const { data } = await axios.get(`/api/v1/dns?find=${encodeURIComponent(f)}`)
					this.domains = Array.isArray(data) ? data : []
				} else {
					const { data } = await axios.get(`/api/v1/src?srcIp=${encodeURIComponent(f)}`)
					this.domains = Array.isArray(data) ? data : []
				}
			} catch (e) {
				this.error = 'Не удалось выполнить запрос. Проверьте доступность API и параметры.'
				console.log(e)
			} finally {
				this.loading = false
			}
		},

		async onToggle(domain, val) {
			domain.isIgnoreVpn = val
			this.error = ''
			try {
				await axios.post(
					`/api/v1/dns?dns=${encodeURIComponent(domain.dstDns)}&enabled=${encodeURIComponent(val)}`
				)
			} catch (e) {
				domain.isIgnoreVpn = !val
				this.error = `Не удалось обновить ${this.label} для домена.`
				console.log(e)
			}
		},
	},

	async mounted() {
		if (this.filterStr) {
			await this.doFilter()
		}
	},
}
</script>

<style scoped>
.card {
	background: rgba(255, 255, 255, 0.78);
	border: 1px solid rgba(20, 18, 40, 0.10);
	border-radius: 18px;
	box-shadow: 0 18px 40px rgba(20, 18, 40, 0.10);
	overflow: hidden;
}

.card-header {
	display: flex;
	justify-content: space-between;
	gap: 16px;
	padding: 18px 18px 14px;
	border-bottom: 1px solid rgba(20, 18, 40, 0.08);
  align-items: center;
}

.card-title { font-weight: 900; letter-spacing: 0.2px; }

.card-desc {
	margin-top: 4px;
	font-size: 13px;
	opacity: 0.70;
	line-height: 1.35;
	max-width: 720px;
}

.status { display: flex; align-items: center; gap: 10px; }

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
  text-wrap: nowrap;
}

.controls {
	display: flex;
	align-items: flex-end;
	gap: 12px;
	padding: 14px 18px 18px;
	border-bottom: 1px solid rgba(20, 18, 40, 0.06);
  flex-wrap: wrap;
}

.field { display: flex; flex-direction: column; gap: 6px; min-width: fit-content; }
.field.grow { flex: 1; min-width: 260px; }
.label { font-size: 12px; font-weight: 800; opacity: 0.75; }

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

.btn:active { transform: translateY(1px); }
.btn:disabled { opacity: 0.5; cursor: not-allowed; }

.btn.primary {
	border-color: rgba(90, 66, 255, 0.35);
	background: linear-gradient(135deg, rgba(90, 66, 255, 0.92), rgba(107, 87, 255, 0.92));
	color: #fff;
	box-shadow: 0 12px 28px rgba(90, 66, 255, 0.25);
}

.btn.primary:hover { box-shadow: 0 14px 30px rgba(90, 66, 255, 0.30); }

.list { padding: 10px 10px 12px; }

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

.row-main { display: flex; flex-direction: column; gap: 6px; min-width: 0; }
.dns { font-weight: 950; letter-spacing: 0.1px; word-break: break-word; }
.chips { display: flex; flex-wrap: wrap; gap: 6px; }

.chip {
	font-size: 12px;
	font-weight: 800;
	padding: 6px 10px;
	border-radius: 999px;
	border: 1px solid rgba(20, 18, 40, 0.08);
	background: rgba(20, 18, 40, 0.04);
	opacity: 0.95;
}

.row-actions { display: flex; align-items: center; gap: 10px; flex-shrink: 0; }
.muted { opacity: 0.65; }
.small { font-size: 12px; }

.empty {
	padding: 18px 18px 22px;
	text-align: center;
	opacity: 0.7;
	font-weight: 800;
}

@media (max-width: 860px) {
	.controls { flex-direction: column; align-items: stretch; }
	.field { min-width: 0; }
	.row { flex-direction: column; align-items: stretch; }
	.row-actions { justify-content: space-between; }
}
</style>
