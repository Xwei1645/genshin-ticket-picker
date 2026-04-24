<script>
	import Icon from '$lib/components/Icon.svelte';
	import SlotMachine from '$lib/components/SlotMachine.svelte';
	import { localConfig } from '$lib/helpers/dataAPI/api-localstore';
	import { removeAnimClass } from '$lib/helpers/transition';

	export let bonusType = 'stardust';
	export let type;
	export let ticketNo = 0;

	const formatTicketNo = (value) => {
		if (!localConfig.get('ticketPadZero')) return `${value}`;
		const maxRaw = parseInt(localConfig.get('ticketNoMax'), 10);
		const maxForDigit = Number.isNaN(maxRaw) ? 999 : Math.max(0, maxRaw);
		const digits = Math.max(3, `${maxForDigit}`.length);
		return `${value}`.padStart(digits, '0');
	};

	$: ticketDisplay = formatTicketNo(ticketNo);
</script>

{#if type !== 'outfit'}
	<div class="starfate anim {bonusType}" use:removeAnimClass>
		<div class="icon">
			<Icon type={bonusType} width="100%" />
		</div>
		<div class="text">
			<span>获奖门票编号</span>
			<div class="ticket-no">
				<SlotMachine value={ticketDisplay} />
			</div>
		</div>
	</div>
{/if}

<style>
	.starfate {
		justify-content: flex-end;
		right: clamp(-1.7rem, -1.6vw, -0.6rem);
		transform: translateY(-50%);
		text-transform: capitalize;
	}
	.starfate > .icon {
		width: clamp(84px, 8.5vw, 142px);
		margin-right: clamp(-71px, -4vw, -42px);
		position: relative;
		z-index: +1;
		flex-shrink: 0;
		transform: scale(1);
		animation-delay: 1.3s !important;
	}

	.starfate > .icon :global(img),
	.starfate > .icon :global(svg) {
		width: 100%;
		height: auto;
	}
	.starfate.anim > .icon {
		animation: starfateIcon forwards 0.4s 1;
		opacity: 0;
	}
	.starfate.starglitter :global(img) {
		filter: drop-shadow(0 0 6px rgba(245, 193, 63, 1));
	}
	.starglitter .text {
		background-image: linear-gradient(to right, rgba(245, 193, 63, 0.9), rgba(245, 193, 63, 0.1));
		color: rgb(255, 255, 77);
	}
	.starfate.stardust :global(img) {
		filter: drop-shadow(0 0 6px rgba(221, 203, 245, 1));
	}
	.stardust .text {
		background-image: linear-gradient(to right, rgba(104, 47, 173, 0.9), rgba(104, 47, 173, 0.1));
		color: rgb(198, 130, 214);
	}
	.starfate .text {
		width: min(50vw, 610px);
		max-width: 50%;
		padding: clamp(8px, 1vw, 12px) clamp(24px, 2.2vw, 40px);
		position: relative;
		z-index: -1;
		animation-delay: 1.3s !important;
		font-size: clamp(1.3rem, 2vw, 2rem);
	}

	.starfate.anim .text {
		opacity: 0;
		animation: starfateText forwards 0.7s 1;
	}

	:global(.mobile) .starfate .text {
		width: min(58vw, 390px);
		font-size: clamp(1.1rem, 3.5vw, 1.5rem);
	}

	:global(.mobile) .starfate > .icon {
		width: clamp(70px, 18vw, 112px);
		margin-right: clamp(-56px, -8vw, -36px);
	}
	.starfate span {
		color: #ddd;
		position: absolute;
		top: -50%;
		left: clamp(18px, 1.8vw, 35px);
		font-size: clamp(1.2rem, 2vw, 2rem);
		font-weight: 600;
		text-shadow: 0 2px 6px rgba(0, 0, 0, 0.75);
	}

	.ticket-no {
		font-size: clamp(2.4rem, 4.2vw, 4.4rem);
		font-weight: 700;
		line-height: 1.1;
		letter-spacing: 0.08em;
		text-shadow: 0 0 0.4rem rgba(0, 0, 0, 0.35);
		margin-left: clamp(14px, 1.5vw, 28px);
	}

	:global(.mobile) .ticket-no {
		font-size: clamp(2rem, 7vw, 3.3rem);
		margin-left: clamp(10px, 3vw, 18px);
	}

	.starfate {
		position: fixed;
		top: 60%;
		display: flex;
		align-items: center;
		width: 1200px;
		max-width: 95%;
		z-index: +1;
	}

	:global(.mobile) .starfate {
		right: clamp(-0.9rem, -4vw, -0.3rem);
	}

	@keyframes starfateIcon {
		30% {
			transform: scale(1.5);
			opacity: 1;
		}
		100% {
			transform: scale(1);
			opacity: 1;
		}
	}

	@keyframes starfateText {
		0% {
			transform: translateX(-10px);
			opacity: 0;
		}
		100% {
			transform: translateX(0px);
			opacity: 1;
		}
	}
</style>
