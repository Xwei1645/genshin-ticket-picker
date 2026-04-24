<script>
	import { getContext, onMount, setContext } from 'svelte';
	import { fade, fly } from 'svelte/transition';
	import { t } from 'svelte-i18n';
	import {
		activeBanner,
		bannerList,
		assets,
		activeVersion,
		wishAmount,
		acquaint,
		intertwined,
		stardust,
		starglitter,
		customData
	} from '$lib/store/app-stores';
	import { localBalance, localConfig } from '$lib/helpers/dataAPI/api-localstore';
	import { APP_TITLE } from '$lib/env';
	import { playSfx } from '$lib/helpers/audio/audio';
	import WISH, { roll } from '$lib/helpers/gacha/Wish';

	// Components
	import Header from './_header.svelte';
	import Footer from './_footer.svelte';
	import OutOfPrimogem from './_out-of-primogem.svelte';
	import BannerItem from './_banner-item.svelte';
	import EpitomizedModal from './epitomized-path/EpitomizedPath.svelte';
	import Meteor from './wish-result/_meteor.svelte';
	import WishResult from './wish-result/WishResult.svelte';

	let rollCount = 0;
	let result = [];
	let WishInstance;

	let type;
	$: nowBanner = $bannerList[$activeBanner] || {};
	$: ({ type } = nowBanner);
	$: bannerType = type || '';
	$: isEvent = bannerType.match(/(event|chronicled)/);
	$: currencyUsed = isEvent ? $intertwined : $acquaint;
	$: isUnlimited = $wishAmount === 'unlimited';

	// Load Wish Configuration When changing banner Version
	const initialWish = async ({ patch, phase }) => {
		if (!patch || !phase) return;
		WishInstance = await WISH.init(patch, phase, $customData);
	};
	onMount(() => activeVersion.subscribe(initialWish));

	const getIndexOfCharBanner = () => {
		const events = $bannerList.filter(({ type }) => type === 'character-event');
		const index = events.findIndex(({ character }) => character === nowBanner.character);
		return index;
	};

	// Epitomized Modal
	let openEpitomized = false;
	const handleEpitomizedModal = () => (openEpitomized = !openEpitomized);
	setContext('handleEpitomizedModal', handleEpitomizedModal);

	// Wish Roller
	let multi = false;
	let rollCost;
	let showConvertModal = false;
	let onWish = getContext('onWish');

	const createUniqueTicketNo = (used = new Set()) => {
		const minRaw = parseInt(localConfig.get('ticketNoMin'), 10);
		const maxRaw = parseInt(localConfig.get('ticketNoMax'), 10);
		const min = Number.isNaN(minRaw) ? 100 : Math.max(0, Math.min(999998, minRaw));
		const max = Number.isNaN(maxRaw) ? 999 : Math.max(min + 1, Math.min(999999, maxRaw));
		const maxUnique = max - min + 1;

		let ticketNo = min;
		if (used.size >= maxUnique) {
			return min + Math.floor(Math.random() * maxUnique);
		}

		do {
			ticketNo = Math.floor(Math.random() * (max - min + 1)) + min;
		} while (used.has(ticketNo));
		used.add(ticketNo);
		return ticketNo;
	};

	const createForcedRarityList = (count) => {
		const fixedMultiRarity = !!localConfig.get('fixedMultiRarity');
		if (!fixedMultiRarity || count < 2) return [];

		const fiveRaw = parseInt(localConfig.get('multiRarity5'), 10);
		const fourRaw = parseInt(localConfig.get('multiRarity4'), 10);

		const fiveCount = Number.isNaN(fiveRaw) ? 0 : Math.max(0, Math.min(count, fiveRaw));
		const fourCount = Number.isNaN(fourRaw)
			? Math.min(1, count - fiveCount)
			: Math.max(0, Math.min(count - fiveCount, fourRaw));
		const threeCount = Math.max(0, count - fiveCount - fourCount);

		const forcedRarities = [
			...Array(fiveCount).fill(5),
			...Array(fourCount).fill(4),
			...Array(threeCount).fill(3)
		];

		for (let i = forcedRarities.length - 1; i > 0; i--) {
			const j = Math.floor(Math.random() * (i + 1));
			[forcedRarities[i], forcedRarities[j]] = [forcedRarities[j], forcedRarities[i]];
		}

		return forcedRarities;
	};

	const doRoll = async (count, bannerToRoll) => {
		rollCount = count;
		multi = count > 1;
		const tmp = [];
		const forcedRarityList = createForcedRarityList(count);

		rollCost = bannerToRoll === 'beginner' ? 8 : count;
		if (!isUnlimited && rollCost > currencyUsed) return (showConvertModal = true);
		const indexOfCharBanner = bannerToRoll === 'character-event' ? getIndexOfCharBanner() : 0;
		onWish.set(true);

		for (let i = 0; i < count; i++) {
			const result = await roll(bannerToRoll, WishInstance, indexOfCharBanner, {
				forcedRarity: forcedRarityList[i]
			});
			tmp.push(result);
		}

		const usedTicketNo = new Set();
		const rolledWithTicketNo = tmp.map((item) => ({
			...item,
			ticketNo: createUniqueTicketNo(usedTicketNo)
		}));

		result = rolledWithTicketNo;
		handleMeteorAnimation();
		if (isUnlimited) return;
		updateMilestones();
		updateFatesBalance(bannerToRoll);
	};
	setContext('doRoll', doRoll);

	const updateFatesBalance = (banner) => {
		const isAcquaint = ['beginner', 'standard'].includes(banner);
		const funds = isAcquaint ? acquaint : intertwined;
		funds.update((n) => {
			const afterUpdate = n - (banner === 'beginner' && rollCount > 1 ? 8 : rollCount);
			localBalance.set(isAcquaint ? 'acquaint' : 'intertwined', afterUpdate);
			return afterUpdate;
		});
	};

	const updateMilestones = () => {
		const update = (type) => {
			const qty = result.reduce((prev, { bonusQty, bonusType }) => {
				return prev + (bonusType === type ? bonusQty : 0);
			}, 0);

			const milestone = type === 'stardust' ? stardust : starglitter;
			milestone.update((n) => {
				const afterUpdate = n + qty;
				localBalance.set(type, afterUpdate);
				return afterUpdate;
			});
		};

		update('starglitter');
		update('stardust');
	};

	// Wish Result Handler
	let skipSplashArt = false;
	let showWishResult = false;
	let showMeteor = false;
	let single = true;
	let radiance = false;
	let meteorStar = 3;

	const closeResult = () => {
		showWishResult = false;
		onWish.set(false);
		if (multi && localConfig.get('skipObtainedAfterSummary')) return;
		checkObtained();
	};
	setContext('closeResult', closeResult);

	const showSplashArt = ({ skip = false } = {}) => {
		skipSplashArt = skip;
		showMeteor = false;
		showWishResult = true;
	};
	setContext('showSplashArt', showSplashArt);

	const handleMeteorAnimation = () => {
		const autoSkip = localConfig.get('autoskip');
		if (autoSkip) return showSplashArt({ skip: true });

		const stars = result.map(({ rarity }) => rarity);
		single = stars.length === 1;
		meteorStar = 3;
		if (stars.includes(4)) meteorStar = 4;
		if (stars.includes(5)) meteorStar = 5;
		const captureStatus = result.map(({ captured = false }) => captured);
		radiance = captureStatus.includes(true);
		showMeteor = true;
	};

	// Modal Convert
	const closeModal = () => {
		playSfx('close');
		showConvertModal = false;
	};
	setContext('closeModal', closeModal);

	const reroll = (amount) => {
		playSfx();
		const multiAmount = bannerType === 'beginner' ? 10 : amount;
		doRoll(multi ? multiAmount : 1, bannerType);
		showConvertModal = false;
	};
	setContext('reroll', reroll);

	// Obtained Bonus
	const countMilestone = (masterless) => {
		return result.reduce((a, { bonusType, bonusQty }) => {
			return a + (bonusType === masterless ? bonusQty : 0);
		}, 0);
	};

	const showObtained = getContext('openObtained');
	const checkObtained = () => {
		const stardustQty = countMilestone('stardust');
		const starglitterQty = countMilestone('starglitter');

		const obtainedItems = [
			{ item: 'stardust', qty: stardustQty },
			{ item: 'starglitter', qty: starglitterQty }
		];

		if (!stardustQty && !starglitterQty) return;
		showObtained(obtainedItems);
	};
</script>

<svelte:head>
	<title>{$t('title', { default: APP_TITLE })}</title>
</svelte:head>

<div class="overlay" in:fade|local />

<div class="wish-container" class:show={showMeteor || showWishResult}>
	<Meteor show={showMeteor} isSingle={single} rarity={meteorStar} {radiance} />
	{#if showWishResult}
		<WishResult list={result} skip={skipSplashArt} />
	{/if}
</div>

<section style="background-image: url('{$assets['wish-background.webp']}');">
	<div class="col top">
		<Header {bannerType} />
	</div>

	<div class="col banner">
		<div class="item">
			<BannerItem />
		</div>

		<div class="col button" in:fly={{ y: 20, duration: 1000 }}>
			<Footer {bannerType} />
		</div>
	</div>
</section>

{#if openEpitomized}
	<EpitomizedModal />
{/if}

{#if showConvertModal}
	<OutOfPrimogem isEventBanner={isEvent} {rollCost} />
{/if}

<style>
	section {
		width: 100%;
		height: 100%;
		display: flex;
		flex-direction: column;
		justify-content: flex-end;
		align-items: center;
		overflow: hidden;
		background-repeat: no-repeat;
		background-position: center;
		background-size: cover;
		background-position: 20%;
	}

	.overlay {
		position: absolute;
		top: 50%;
		left: 50%;
		transform: translate(-50%, -50%);
		display: block;
		width: 100%;
		height: 100%;
		background-color: rgba(0, 0, 0, 0.08);
		box-shadow: 0 0 50vh rgba(0, 0, 0, 0.4) inset;
	}

	.wish-container {
		position: fixed;
		width: 100%;
		height: 100%;
		z-index: 15;
		top: 0;
		left: 0;
		pointer-events: none;
	}

	.wish-container.show {
		pointer-events: unset;
	}

	.top,
	.banner,
	.button,
	.item {
		display: block;
		width: 100%;
	}

	.top {
		min-height: 70px;
	}
	.banner,
	.item {
		height: 100%;
	}
	.item {
		position: relative;
	}
	.banner {
		display: flex;
		justify-content: center;
		flex-direction: column;
	}

	.button {
		height: 120px;
	}

	/* Mobile */
	:global(.mobile) section {
		flex-direction: row;
	}
	:global(.mobile) .top {
		height: 100%;
		width: min-content;
	}
	:global(.mobile) .banner {
		width: 120%;
		margin-left: -20px;
	}
	:global(.mobile) .button {
		height: 50px;
		margin-bottom: 0.2rem;
	}
</style>
