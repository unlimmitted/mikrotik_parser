<template>
	<div v-if="message" class="alert">
		<div class="alert-title">Ошибка</div>
		<div class="alert-text">{{ message }}</div>
	</div>
</template>

<script>
export default {
	name: 'AppAlert',
	props: {
		message: {
			type: String,
			default: '',
		},
	},
}
</script>

<style scoped>
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
</style>
