<script>
	import { t } from 'svelte-i18n';

	import { wishAmount } from '$lib/store/app-stores';
	import { localConfig } from '$lib/helpers/dataAPI/api-localstore';
	import CheckBox from '$lib/components/CheckBox.svelte';

	const clamp = (value, min, max) => {
		if (Number.isNaN(value)) return min;
		return Math.max(min, Math.min(max, value));
	};

	const parseNumber = (value, fallback) => {
		const parsed = parseInt(value, 10);
		return Number.isNaN(parsed) ? fallback : parsed;
	};

	let hideWishExtras = !!localConfig.get('hideWishExtras');
	let skipObtainedAfterSummary = !!localConfig.get('skipObtainedAfterSummary');
	let fixedMultiRarity = !!localConfig.get('fixedMultiRarity');
	let unlimitedCurrency = localConfig.get('wishAmount') === 'unlimited';
	let ticketPadZero = !!localConfig.get('ticketPadZero');
	let customWatermarkText = `${localConfig.get('customWatermarkText') || ''}`;
	let watermarkUseSystemSans = !!localConfig.get('watermarkUseSystemSans');

	let multiRarity5 = clamp(parseNumber(localConfig.get('multiRarity5'), 0), 0, 10);
	let multiRarity4 = clamp(parseNumber(localConfig.get('multiRarity4'), 1), 0, 10 - multiRarity5);

	let ticketNoMin = clamp(parseNumber(localConfig.get('ticketNoMin'), 100), 0, 999998);
	let ticketNoMax = clamp(parseNumber(localConfig.get('ticketNoMax'), 999), ticketNoMin + 1, 999999);

	$: threeStarCount = Math.max(0, 10 - multiRarity5 - multiRarity4);

	const toggleConfig = (key, checked) => {
		localConfig.set(key, !!checked);
	};

	const handleUnlimitedCurrency = ({ detail }) => {
		unlimitedCurrency = !!detail.checked;
		const amountMode = unlimitedCurrency ? 'unlimited' : 'default';
		localConfig.set('wishAmount', amountMode);
		wishAmount.set(amountMode);
	};

	const saveRarityConfig = () => {
		multiRarity5 = clamp(parseNumber(multiRarity5, 0), 0, 10);
		multiRarity4 = clamp(parseNumber(multiRarity4, 0), 0, 10 - multiRarity5);
		localConfig.set('multiRarity5', multiRarity5);
		localConfig.set('multiRarity4', multiRarity4);
	};

	const saveTicketRange = () => {
		ticketNoMin = clamp(parseNumber(ticketNoMin, 100), 0, 999998);
		ticketNoMax = clamp(parseNumber(ticketNoMax, 999), ticketNoMin + 1, 999999);
		localConfig.set('ticketNoMin', ticketNoMin);
		localConfig.set('ticketNoMax', ticketNoMax);
	};

	const saveWatermarkText = () => {
		const trimmed = `${customWatermarkText || ''}`.trimStart().slice(0, 40);
		customWatermarkText = trimmed;
		localConfig.set('customWatermarkText', trimmed);
	};
</script>

<div class="content-container">
	<h2>{$t('menu.wishDrawSettings')}</h2>

	<div class="card">
		<CheckBox
			id="hide-extra"
			checked={hideWishExtras}
			on:change={({ detail }) => {
				hideWishExtras = !!detail.checked;
				toggleConfig('hideWishExtras', hideWishExtras);
			}}
		>
			{$t('menu.hideWishExtras')}
		</CheckBox>

		<CheckBox
			id="skip-obtained"
			checked={skipObtainedAfterSummary}
			on:change={({ detail }) => {
				skipObtainedAfterSummary = !!detail.checked;
				toggleConfig('skipObtainedAfterSummary', skipObtainedAfterSummary);
			}}
		>
			{$t('menu.skipObtainedAfterSummary')}
		</CheckBox>

		<CheckBox id="unlimited-currency" checked={unlimitedCurrency} on:change={handleUnlimitedCurrency}>
			{$t('menu.unlimitedCurrencyMode')}
		</CheckBox>
	</div>

	<h2>{$t('menu.tenPullRarityPreset')}</h2>
	<div class="card">
		<CheckBox
			id="fixed-rarity"
			checked={fixedMultiRarity}
			on:change={({ detail }) => {
				fixedMultiRarity = !!detail.checked;
				toggleConfig('fixedMultiRarity', fixedMultiRarity);
			}}
		>
			{$t('menu.enableTenPullRarityPreset')}
		</CheckBox>

		<div class="input-row" class:disabled={!fixedMultiRarity}>
			<label for="rarity5">{$t('menu.tenPullFiveStarCount')}</label>
			<input
				id="rarity5"
				type="number"
				min="0"
				max="10"
				disabled={!fixedMultiRarity}
				bind:value={multiRarity5}
				on:input={saveRarityConfig}
			/>
		</div>

		<div class="input-row" class:disabled={!fixedMultiRarity}>
			<label for="rarity4">{$t('menu.tenPullFourStarCount')}</label>
			<input
				id="rarity4"
				type="number"
				min="0"
				max="10"
				disabled={!fixedMultiRarity}
				bind:value={multiRarity4}
				on:input={saveRarityConfig}
			/>
		</div>

		<div class="preview">{$t('menu.tenPullThreeStarCount', { values: { count: threeStarCount } })}</div>
	</div>

	<h2>{$t('menu.ticketRange')}</h2>
	<div class="card range">
		<CheckBox
			id="ticket-pad-zero"
			checked={ticketPadZero}
			on:change={({ detail }) => {
				ticketPadZero = !!detail.checked;
				toggleConfig('ticketPadZero', ticketPadZero);
			}}
		>
			{$t('menu.ticketPadZero')}
		</CheckBox>

		<div class="input-row">
			<label for="ticket-min">{$t('menu.ticketMin')}</label>
			<input id="ticket-min" type="number" min="0" max="999998" bind:value={ticketNoMin} on:input={saveTicketRange} />
		</div>

		<div class="input-row">
			<label for="ticket-max">{$t('menu.ticketMax')}</label>
			<input id="ticket-max" type="number" min="1" max="999999" bind:value={ticketNoMax} on:input={saveTicketRange} />
		</div>
	</div>

	<h2>{$t('menu.watermarkReplace')}</h2>
	<div class="card range">
		<CheckBox
			id="watermark-system-sans"
			checked={watermarkUseSystemSans}
			on:change={({ detail }) => {
				watermarkUseSystemSans = !!detail.checked;
				toggleConfig('watermarkUseSystemSans', watermarkUseSystemSans);
			}}
		>
			{$t('menu.watermarkUseSystemSans')}
		</CheckBox>

		<div class="input-row">
			<label for="custom-watermark">{$t('menu.watermarkCustomText')}</label>
			<input
				id="custom-watermark"
				type="text"
				maxlength="40"
				placeholder={$t('menu.watermarkDefaultPlaceholder')}
				bind:value={customWatermarkText}
				on:input={saveWatermarkText}
			/>
		</div>
	</div>
</div>

<style>
	.content-container {
		overflow-y: auto;
		padding-right: 0.35rem;
	}

	.card {
		background: #f2ece0;
		padding: 0.7rem 1rem;
		border-radius: 0.7rem;
		margin-bottom: 0.8rem;
	}

	.input-row {
		display: flex;
		justify-content: space-between;
		align-items: center;
		gap: 1rem;
		padding: 0.35rem 0;
	}

	.input-row.disabled {
		opacity: 0.5;
	}

	label {
		font-size: 0.85rem;
		color: #3f4657;
	}

	input {
		width: 7rem;
		max-width: 45%;
		padding: 0.3rem 0.6rem;
		border-radius: 999px;
		border: 1px solid #b8ad99;
		background: #fff;
		text-align: center;
	}

	.preview {
		font-size: 0.8rem;
		color: #52596a;
		padding: 0.2rem 0.1rem;
	}

	.range {
		padding-bottom: 0.9rem;
	}

	:global(.checkbox) {
		margin: 0.5rem 0;
	}
</style>
