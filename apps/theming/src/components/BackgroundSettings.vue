<!--
  - @copyright Copyright (c) 2020 Julius Härtl <jus@bitgrid.net>
  - @copyright Copyright (c) 2022 Greta Doci <gretadoci@gmail.com>
  -
  - @author Julius Härtl <jus@bitgrid.net>
  - @author Greta Doci <gretadoci@gmail.com>
  - @author Christopher Ng <chrng8@gmail.com>
  -
  - @license GNU AGPL version 3 or any later version
  -
  - This program is free software: you can redistribute it and/or modify
  - it under the terms of the GNU Affero General Public License as
  - published by the Free Software Foundation, either version 3 of the
  - License, or (at your option) any later version.
  -
  - This program is distributed in the hope that it will be useful,
  - but WITHOUT ANY WARRANTY; without even the implied warranty of
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program. If not, see <http://www.gnu.org/licenses/>.
  -
  -->

<template>
	<div class="background-selector">
		<button class="background filepicker"
			:class="{ active: background === 'custom' }"
			tabindex="0"
			@click="pickFile">
			{{ t('theming', 'Pick from Files') }}
		</button>
		<button class="background default"
			tabindex="0"
			:class="{ 'icon-loading': loading === 'default', active: background === 'default' }"
			@click="setDefault">
			{{ t('theming', 'Default image') }}
		</button>
		<button class="background color"
			:class="{ active: background === 'custom' }"
			tabindex="0"
			@click="pickColor">
			{{ t('theming', 'Plain background') }}
		</button>
		<button v-for="shippedBackground in shippedBackgrounds"
			:key="shippedBackground.name"
			v-tooltip="shippedBackground.details.attribution"
			:class="{ 'icon-loading': loading === shippedBackground.name, active: background === shippedBackground.name }"
			tabindex="0"
			class="background"
			:style="{ 'background-image': 'url(' + shippedBackground.preview + ')' }"
			@click="setShipped(shippedBackground.name)" />
	</div>
</template>

<script>
import axios from '@nextcloud/axios'
import Tooltip from '@nextcloud/vue/dist/Directives/Tooltip'
import { generateUrl } from '@nextcloud/router'
import { loadState } from '@nextcloud/initial-state'
import { getBackgroundUrl } from '../helpers/getBackgroundUrl.js'
import { prefixWithBaseUrl } from '../helpers/prefixWithBaseUrl.js'

const shippedBackgroundList = loadState('theming', 'shippedBackgrounds')

export default {
	name: 'BackgroundSettings',
	directives: {
		Tooltip,
	},
	props: {
		background: {
			type: String,
			default: 'default',
		},
		themingDefaultBackground: {
			type: String,
			default: '',
		},
	},
	data() {
		return {
			backgroundImage: generateUrl('/apps/theming/background') + '?v=' + Date.now(),
			loading: false,
		}
	},
	computed: {
		shippedBackgrounds() {
			return Object.keys(shippedBackgroundList).map(fileName => {
				return {
					name: fileName,
					url: prefixWithBaseUrl(fileName),
					preview: prefixWithBaseUrl('preview/' + fileName),
					details: shippedBackgroundList[fileName],
				}
			})
		},
	},
	methods: {
		async update(data) {
			const background = data.type === 'custom' || data.type === 'default' ? data.type : data.value
			this.backgroundImage = getBackgroundUrl(background, data.version, this.themingDefaultBackground)
			if (data.type === 'color' || (data.type === 'default' && this.themingDefaultBackground === 'backgroundColor')) {
				this.$emit('update:background', data)
				this.loading = false
				return
			}
			const image = new Image()
			image.onload = () => {
				this.$emit('update:background', data)
				this.loading = false
			}
			image.src = this.backgroundImage
		},
		async setDefault() {
			this.loading = 'default'
			const result = await axios.post(generateUrl('/apps/theming/background/default'))
			this.update(result.data)
		},
		async setShipped(shipped) {
			this.loading = shipped
			const result = await axios.post(generateUrl('/apps/theming/background/shipped'), { value: shipped })
			this.update(result.data)
		},
		async setFile(path) {
			this.loading = 'custom'
			const result = await axios.post(generateUrl('/apps/theming/background/custom'), { value: path })
			this.update(result.data)
		},
		async pickColor() {
			this.loading = 'color'
			const color = OCA && OCA.Theming ? OCA.Theming.color : '#0082c9'
			const result = await axios.post(generateUrl('/apps/theming/background/color'), { value: color })
			this.update(result.data)
		},
		pickFile() {
			window.OC.dialogs.filepicker(t('theming', 'Insert from {productName}', { productName: OC.theme.name }), (path, type) => {
				if (type === OC.dialogs.FILEPICKER_TYPE_CHOOSE) {
					this.setFile(path)
				}
			}, false, ['image/png', 'image/gif', 'image/jpeg', 'image/svg'], true, OC.dialogs.FILEPICKER_TYPE_CHOOSE)
		},
	},
}
</script>

<style scoped lang="scss">
.background-selector {
	display: flex;
	flex-wrap: wrap;
	justify-content: center;

	.background {
		width: 176px;
		height: 96px;
		margin: 8px;
		background-size: cover;
		background-position: center center;
		text-align: center;
		border-radius: var(--border-radius-large);
		border: 2px solid var(--color-main-background);
		overflow: hidden;

		&.current {
			background-image: var(--color-background-dark);
		}

		&.filepicker, &.default, &.color {
			border-color: var(--color-border);
		}

		&.color {
			background-color: var(--color-primary);
			color: var(--color-primary-text);
		}

		&.active,
		&:hover,
		&:focus {
			border: 2px solid var(--color-primary);
		}

		&.active:not(.icon-loading):after {
			background-image: var(--icon-checkmark-white);
			background-repeat: no-repeat;
			background-position: center;
			background-size: 44px;
			content: '';
			display: block;
			height: 100%;
		}
	}
}
</style>
