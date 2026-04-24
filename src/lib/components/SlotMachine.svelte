<script>
	import { onMount, tick } from 'svelte';

	export let value = ''; // 传入的编号字符串，如 "007"
	export let duration = 2000; // 动画总时长
	export let delay = 1300; // 延迟开始时间，与父组件动画同步

	let digits = [];
	$: {
		digits = value.split('').map((d, i) => ({
			target: d,
			currentOffset: 0,
			isSpinning: false,
			delay: i * 150 // 每个数字稍微有一点偏移，更有节奏感
		}));
	}

	const numbers = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 0, 1, 2, 3, 4, 5, 6, 7, 8, 9];

	async function startSpin() {
		await new Promise((resolve) => setTimeout(resolve, delay));
		digits = digits.map((d, i) => ({ ...d, isSpinning: true }));
	}

	onMount(() => {
		startSpin();
	});
</script>

<div class="slot-machine">
	{#each digits as digit, i (i)}
		<div class="digit-container">
			<div
				class="digit-track"
				class:spinning={digit.isSpinning}
				style="--target: {digit.target}; --delay: {digit.delay}ms; --duration: {duration}ms"
			>
				<div class="digit-item">0</div>
				{#each numbers as n}
					<div class="digit-item">{n}</div>
				{/each}
				<div class="digit-item target">{digit.target}</div>
			</div>
		</div>
	{/each}
</div>

<style>
	.slot-machine {
		display: inline-flex;
		height: 1.1em;
		align-items: center;
		/* 使用 mask 实现上下边缘的渐变模糊/隐藏效果 */
		-webkit-mask-image: linear-gradient(
			to bottom,
			transparent 0%,
			black 20%,
			black 80%,
			transparent 100%
		);
		mask-image: linear-gradient(
			to bottom,
			transparent 0%,
			black 20%,
			black 80%,
			transparent 100%
		);
	}

	.digit-container {
		position: relative;
		width: 0.8em;
		height: 1.1em;
		overflow: hidden;
		margin: 0 0.02em;
	}

	.digit-track {
		display: flex;
		flex-direction: column;
		transform: translateY(0);
		text-align: center;
	}

	.digit-track.spinning {
		animation: spin var(--duration) cubic-bezier(0.1, 0.6, 0.2, 1) var(--delay) forwards;
	}

	.digit-item {
		height: 1.1em;
		display: flex;
		align-items: center;
		justify-content: center;
		flex-shrink: 0;
	}

	@keyframes spin {
		0% {
			transform: translateY(0);
		}
		100% {
			/* 修正偏移：21步（2组0-9+1个target） */
			transform: translateY(calc(-1.1em * 21));
		}
	}
</style>
