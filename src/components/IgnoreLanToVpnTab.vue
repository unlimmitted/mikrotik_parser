<template>
	<section class="card">
		<div class="card-header">
			<div>
				<div class="card-title">{{ label }}</div>
				<div class="card-desc">
					Ввод внутреннего адреса/подсети (IP или CIDR), которые нужно добавить в {{ label }}
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
					@keydown.enter.prevent="addLanIgnore"
				/>
			</div>

			<div class="field">
				<label class="label">Фильтр</label>
				<SegmentedControl v-model="lanFilter" :options="filterOptions" />
			</div>

			<button class="btn primary" @click="addLanIgnore" :disabled="!canAdd">
				Добавить
			</button>
		</div>

		<AppAlert :message="lanError" />

		<div class="list">
			<div v-for="item in filteredList" :key="item.ip" class="row">
				<div class="row-main">
					<div class="dns">{{ item.hostName || '—' }}</div>
					<div class="muted small">{{ item.ip }}</div>
				</div>

				<div class="row-actions">
					<ToggleSwitch
						:input-id="`lan-${item.ip}`"
						:model-value="item.enabled"
						:label="label"
						@update:modelValue="val => onToggle(item, val)"
					/>
				</div>
			</div>

			<div v-if="!loadingLan && lanIgnoreList.length === 0" class="empty">
				Пока нет адресов. Добавьте IP или подсеть.
			</div>
			<div v-else-if="!loadingLan && filteredList.length === 0" class="empty">
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
	name: 'IgnoreLanToVpnTab',
	components: { AppAlert, SegmentedControl, ToggleSwitch },

	props: {
		label: { type: String, default: 'ignoreLanToVpn' },
	},

	data() {
		return {
			lanIgnoreInput: '',
			lanIgnoreList: [],
			lanError: '',
			lanFilter: 'all',
			loadingLan: false,

			filterOptions: [
				{ value: 'all', label: 'все' },
				{ value: 'enabled', label: 'включенные' },
				{ value: 'disabled', label: 'отключенные' },
			],
		}
	},

	computed: {
		canAdd() {
			return !!(this.lanIgnoreInput && this.lanIgnoreInput.trim().length > 0)
		},
		filteredList() {
			const list = Array.isArray(this.lanIgnoreList) ? this.lanIgnoreList : []
			if (this.lanFilter === 'enabled') return list.filter(x => x.enabled === true)
			if (this.lanFilter === 'disabled') return list.filter(x => x.enabled === false)
			return list
		},
	},

	methods: {
		async fetchLanIgnoreList() {
			this.lanError = ''
			this.loadingLan = true
			try {
				const { data } = await axios.get('/api/v1/ignore-lan-to-vpn')
				this.lanIgnoreList = Array.isArray(data)
					? data
						.map(x => ({
							ip: String(x.ip ?? '').trim(),
							hostName: String(x.hostName ?? '').trim(),
							enabled: x.enabled === true,
							dynamic: x.dynamic === true,
						}))
						.filter(x => x.ip.length > 0)
					: []
			} catch (e) {
				this.lanError = `Не удалось загрузить список ${this.label}.`
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
				await axios.post('/api/v1/ignore-lan-to-vpn', { ip, enabled: true, hostName: '' })
				this.lanIgnoreInput = ''
				await this.fetchLanIgnoreList()
			} catch (e) {
				this.lanError = `Не удалось добавить адрес в ${this.label}.`
				console.log(e)
			}
		},

		async onToggle(item, val) {
			item.enabled = val
			this.lanError = ''
			const prev = !val
			try {
				await axios.post('/api/v1/ignore-lan-to-vpn', {
					ip: item.ip,
					enabled: val,
					hostName: item.hostName,
				})
			} catch (e) {
				item.enabled = prev
				this.lanError = `Не удалось обновить ${this.label}.`
				console.log(e)
			}
		},

		isIpOrCidr(value) {
			const ip = '(?:25[0-5]|2[0-4]\\d|1\\d\\d|[1-9]?\\d)'
			const ipRegex = new RegExp(`^${ip}\\.${ip}\\.${ip}\\.${ip}$`)
			const cidrRegex = new RegExp(`^${ip}\\.${ip}\\.${ip}\\.${ip}\\/(?:[0-9]|[1-2]\\d|3[0-2])$`)
			return ipRegex.test(value) || cidrRegex.test(value)
		},
	},

	async mounted() {
		await this.fetchLanIgnoreList()
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
}

.controls {
	display: flex;
	align-items: flex-end;
	gap: 12px;
	padding: 14px 18px 18px;
	border-bottom: 1px solid rgba(20, 18, 40, 0.06);
}

.field { display: flex; flex-direction: column; gap: 6px; min-width: 220px; }
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
