<template>
	<div>
		<label v-if="label" :for="id" class="hidden-visually">{{ label }}</label>
		<NcSelect :value="inputValue"
			:options="groupsArray"
			:placeholder="placeholder || label"
			:filter-by="filterGroups"
			:input-id="id"
			:limit="5"
			label="displayname"
			:multiple="true"
			:close-on-select="false"
			:disabled="disabled"
			@input="update"
			@search="onSearch" />
	</div>
</template>

<script>
import NcSelect from '@nextcloud/vue/dist/Components/NcSelect.js'

import axios from '@nextcloud/axios'
import { showError } from '@nextcloud/dialogs'
import { generateOcsUrl } from '@nextcloud/router'
import { debounce } from 'debounce'

export default {
	name: 'SettingsSelectGroup',
	components: {
		NcSelect,
	},
	props: {
		/**
		 * The text of the label element of the select group input
		 */
		label: {
			type: String,
			required: true,
		},

		/**
		 * Placeholder for the input element
		 * For backwards compatibility it falls back to the `label` value
		 */
		placeholder: {
			type: String,
			default: '',
		},

		/**
		 * id attribute of the select group element
		 */
		id: {
			type: String,
			default: () => 'action-GenRandomId',
			validator: id => id.trim() !== '',
		},

		/**
		 * value of the select group input
		 * A list of group IDs can be provided
		 */
		value: {
			type: Array,
			default: () => [],
		},

		/**
		 * disabled state of the settings select group input
		 */
		disabled: {
			type: Boolean,
			default: false,
		},
	},
	emits: [
		'input',
		'error',
	],
	data() {
		return {
			/** Temporary store to cache groups */
			groups: {},
			randId: 'GenRandomId',
		}
	},
	computed: {
		/**
		 * Validate input value and only return valid strings (group IDs)
		 *
		 * @return {string[]}
		 */
		filteredValue() {
			return this.value.filter((group) => group !== '' && typeof group === 'string')
		},

		/**
		 * value property converted to an array of group objects used as input for the NcSelect
		 */
		inputValue() {
			return this.filteredValue.map(
				(id) => {
					if (typeof this.groups[id] === 'undefined') {
						return {
							id,
							displayname: id,
						}
					}
					return this.groups[id]
				},
			)
		},

		/**
		 * Convert groups object to array of groups required for NcSelect.options
		 * Filter out currently selected values
		 *
		 * @return {object[]}
		 */
		groupsArray() {
			return Object.values(this.groups).filter(g => !this.value.includes(g.id))
		},
	},
	watch: {
		/**
		 * If the value is changed, check that all groups are loaded so we show the correct display name
		 */
		value: {
			handler() {
				const loadedGroupIds = Object.keys(this.groups)
				const missing = this.filteredValue.filter(group => !loadedGroupIds.includes(group))
				missing.forEach((groupId) => {
					this.loadGroup(groupId)
				})
			},
			// Run the watch handler also when the component is initially mounted
			immediate: true,
		},
	},
	/**
	 * Load groups matching the empty query to reduce API calls
	 */
	async mounted() {
		// version scoped to prevent issues with different library versions
		const storageName = 'announcementcenter/initialGroups'

		let savedGroups = window.sessionStorage.getItem(storageName)
		if (savedGroups) {
			savedGroups = Object.fromEntries(JSON.parse(savedGroups).map(group => [group.id, group]))
			this.groups = { ...this.groups, ...savedGroups }
		} else {
			await this.loadGroup('')
			window.sessionStorage.setItem(storageName, JSON.stringify(Object.values(this.groups)))
		}
	},
	methods: {
		/**
		 * Called when a new group is selected or previous group is deselected to emit the update event
		 *
		 * @param {object[]} updatedValue Array of selected groups
		 */
		update(updatedValue) {
			const value = updatedValue.map((element) => element.id)
			/** Emitted when the groups selection changes<br />**Payload:** `value` (`Array`) - *Ids of selected groups */
			this.$emit('input', value)
		},

		/**
		 * Use provisioning API to search for given group and save it in the groups object
		 *
		 * @param {string} query The query like parts of the id oder display name
		 * @return {boolean}
		 */
		async loadGroup(query) {
			try {
				query = typeof query === 'string' ? encodeURI(query) : ''
				const response = await axios.get(generateOcsUrl(`cloud/groups/details?search=${query}&limit=10`, 2))

				if (Object.keys(response.data.ocs.data.groups).length > 0) {
					const newGroups = Object.fromEntries(response.data.ocs.data.groups.map((element) => [element.id, element]))
					this.groups = { ...this.groups, ...newGroups }
					return true
				}
			} catch (error) {
				/** Emitted if groups could not be queried.<br />**Payload:** `error` (`object`) - The Axios error */
				this.$emit('error', error)
				showError(t('Unable to search the group'))
			}
			return false
		},

		/**
		 * Custom filter function for `NcSelect` to filter by ID *and* display name
		 *
		 * @param {object} option One of the groups
		 * @param {string} label The label property of the group
		 * @param {string} search The current search string
		 */
		filterGroups(option, label, search) {
			return `${label || ''} ${option.id}`.toLocaleLowerCase().indexOf(search.toLocaleLowerCase()) > -1
		},

		/**
		 * Debounce the group search (reduce API calls)
		 */
		onSearch: debounce(function(query) {
			this.loadGroup(query)
		}, 200),
	},
}
</script>
