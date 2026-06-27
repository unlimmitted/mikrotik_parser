<template>
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
				@click="$emit('update:activeTab', 'ignoreVPN')"
			>
				{{ ignoreVPNLabel }}
			</button>
			<button
				class="tab"
				:class="{ active: activeTab === 'ignoreLanToVPN' }"
				role="tab"
				:aria-selected="activeTab === 'ignoreLanToVPN'"
				type="button"
				@click="$emit('update:activeTab', 'ignoreLanToVPN')"
			>
				{{ ignoreLanToVPNLabel }}
			</button>
		</nav>
	</header>
</template>

<script>
export default {
	name: 'AppTopbar',
	props: {
		activeTab: { type: String, required: true },
		ignoreVPNLabel: { type: String, default: 'ignoreVpn' },
		ignoreLanToVPNLabel: { type: String, default: 'ignoreLanToVpn' },
	},
	emits: ['update:activeTab'],
}
</script>

<style scoped>
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
  width: 100%;
}

.tab:hover { background: rgba(90, 66, 255, 0.08); }
.tab:active { transform: translateY(1px); }

.tab.active {
	background: white;
	color: #1b1630;
	box-shadow: 0 8px 18px rgba(20, 18, 40, 0.10);
	border: 1px solid rgba(20, 18, 40, 0.08);
}

@media (max-width: 860px) {
	.topbar { flex-direction: column; align-items: stretch; }
	.brand { min-width: 0; }
}
</style>
