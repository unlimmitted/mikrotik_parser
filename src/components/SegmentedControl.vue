<template>
	<div class="segmented">
		<button
			v-for="option in options"
			:key="option.value"
			class="seg"
			:class="{ active: modelValue === option.value }"
			type="button"
			@click="$emit('update:modelValue', option.value)"
		>
			{{ option.label }}
		</button>
	</div>
</template>

<script>
export default {
	name: 'SegmentedControl',
	props: {
		modelValue: {
			type: String,
			required: true,
		},
		options: {
			// [{ value: 'byDomain', label: 'по домену' }, ...]
			type: Array,
			required: true,
		},
	},
	emits: ['update:modelValue'],
}
</script>

<style scoped>
.segmented {
	display: flex;
	gap: 6px;
	padding: 6px;
	border-radius: 14px;
	background: rgba(20, 18, 40, 0.04);
	border: 1px solid rgba(20, 18, 40, 0.08);
  min-width: fit-content;
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
  text-wrap: nowrap;
  width: 100%;
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
</style>
