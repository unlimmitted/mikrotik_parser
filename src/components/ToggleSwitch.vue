<template>
	<div class="toggle">
		<input
			class="toggle-input"
			type="checkbox"
			:id="inputId"
			:checked="modelValue"
			@change="$emit('update:modelValue', $event.target.checked)"
		/>
		<label class="toggle-label" :for="inputId">
			<span class="toggle-track"></span>
			<span class="toggle-text">{{ label }}</span>
		</label>
	</div>
</template>

<script>
export default {
	name: 'ToggleSwitch',
	props: {
		modelValue: {
			type: Boolean,
			default: false,
		},
		inputId: {
			type: String,
			required: true,
		},
		label: {
			type: String,
			default: '',
		},
	},
	emits: ['update:modelValue'],
}
</script>

<style scoped>
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
</style>
