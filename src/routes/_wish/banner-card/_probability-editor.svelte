<script>
	import { getContext } from 'svelte';
	import { fade, fly } from 'svelte/transition';
	import { t } from 'svelte-i18n';
	import { activeBanner, bannerList, chronicledCourse } from '$lib/store/app-stores';
	import { localPity } from '$lib/helpers/dataAPI/api-localstore';
	import { getChronicledRate, getRate, setRate } from '$lib/helpers/gacha/probabilities';
	import { playSfx } from '$lib/helpers/audio/audio';
	import { get5StarItem, regionElement } from '$lib/helpers/gacha/itemdrop-base';
	import ButtonGeneral from '$lib/components/ButtonGeneral.svelte';
	import ButtonModal from '$lib/components/ButtonModal.svelte';

	export let element = 'default';

	const { type: banner, region, stdver, characters: ch, weapons: wp } = $bannerList[$activeBanner];
	const type = banner || 'character-event';
	const isChronicled = type === 'chronicled';
	element = isChronicled ? regionElement(region) : element;

	const editprob = getContext('editprob');
	const showModalReset = getContext('showModalReset');
	const resetClick = () => {
		showModalReset();
		playSfx();
	};

	const targetRate = (banner) => {
		if (banner !== 'chronicled') return getRate(type, 'selectedRate');
		const { type: itemType } = $chronicledCourse;
		const droplist = get5StarItem({
			banner: 'chronicled',
			region: region,
			stdver: stdver,
			rateupItem: itemType === 'weapon' ? wp['5star'] : ch['5star'],
			type: itemType
		});
		const { targetRate } = getChronicledRate(droplist);
		return targetRate;
	};

	$: baseRate4 = getRate(type, 'baseRate4');
	$: baseRate5 = getRate(type, 'baseRate5');
	$: charRate = getRate(type, 'charRate');
	$: winRate = getRate(type, 'winRate');
	$: selectedRate = targetRate(type);
	$: hard4 = getRate(type, 'hard4');
	$: hard5 = getRate(type, 'hard5');
	$: max4 = getRate(type, 'max4');
	$: max5 = getRate(type, 'max5');
	$: guaranteed = getRate(type, 'guaranteed') || 'default';

	let showGuarateedOption = false;
	const guaranteedToggle = (selected) => {
		setRate(type, 'guaranteed', selected);
		guaranteed = selected;
		showGuarateedOption = false;
		playSfx('click2');
	};

	const changeRate = (e, variable) => {
		const val = e.target.value;
		const maxVal = variable.match('Rate4') ? 100 - baseRate5 : 100;
		const capVal = parseFloat(val) < maxVal || !val ? val : maxVal;

		if (val >= maxVal) e.target.value = capVal;
		setRate(type, variable, capVal || 0);

		if (variable === 'winRate') return (winRate = capVal);
		if (variable === 'baseRate4') return (baseRate4 = capVal);

		if (variable === 'baseRate5') {
			baseRate5 = capVal;
			const isOverLimit = parseFloat(baseRate5) + parseFloat(baseRate4) > 100;
			if (!isOverLimit) return;
			baseRate4 = 100 - baseRate5;
			setRate(type, 'baseRate4', baseRate4);
		}
	};

	const changePity = (e, variable) => {
		const val = e.target.value;
		let finalVal = 1;

		// Max Pity Changer
		if (variable.match('max')) {
			const capVal = val <= 1 || !val ? 1 : val;
			finalVal = parseInt(capVal);
			e.target.value = finalVal;

			if (variable === 'max5') {
				max5 = finalVal;
				const maxHard = hard5 >= max5 - 1 ? max5 - 1 : hard5;
				if (hard5 >= maxHard || isNaN(finalVal)) {
					hard5 = isNaN(finalVal) ? 0 : maxHard;
					setRate(type, 'hard5', hard5 || 0);
				}
			}

			if (variable === 'max4') {
				max4 = finalVal;
				const maxHard = hard4 >= max4 - 1 ? max4 - 1 : hard4;
				if (hard4 >= maxHard || isNaN(finalVal)) {
					hard4 = isNaN(finalVal) ? 0 : maxHard;
					setRate(type, 'hard4', hard4 || 0);
				}
			}
		}

		// Hard Pity Changer
		if (variable.match('hard')) {
			const maxPity = variable === 'hard5' ? max5 : max4;
			const cap = (maxPity || 0) - 1;
			finalVal = val >= cap ? cap : val;
			if (val >= cap) e.target.value = finalVal;
			if (variable === 'hard4') hard4 = finalVal;
			if (variable === 'hard5') hard5 = finalVal;
		}

		if (!variable.match('now')) return setRate(type, variable, finalVal || 1);

		// Current Pity Changer
		const value = parseInt(val);
		if (variable === 'now4') localPity.set(`pity4-${type}`, value);
		if (variable === 'now5') localPity.set(`pity5-${type}`, value);
	};
</script>
