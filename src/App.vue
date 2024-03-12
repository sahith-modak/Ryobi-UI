<template>
	<div>
		<header class="header">
			<div class="logo">
				<img
					src="/logo-ryobi-black.svg"
					alt="Logo" />
			</div>
			<nav class="menu"></nav>
		</header>
		<div
			class="container"
			:class="showChat ? 'open-container' : 'close-container'">
			<div
				class="chat-header"
				@click="
					() => {
						getChatHistory(), (showChat = true)
					}
				"
				:style="showChat ? 'cursor:default' : 'cursor:pointer'">
				<div
					class="roboto-medium"
					style="paddingleft: 12px">
					Message
				</div>
				<span
					class="material-symbols-outlined"
					style="paddingright: 12px; cursor: pointer"
					v-if="showChat"
					@click.stop="() => (showChat = false)">
					remove
				</span>
			</div>
			<deep-chat
				v-show="showChat"
				:style="styles"
				:messageStyles="messageStyles"
				:textInput="textInputStyles"
				:htmlClassUtilities="htmlClassUtilities"
				:submitButtonStyles="submitButtonStyles"
				:request="{
					websocket: true,
					handler: handler
				}"
				:requestInterceptor="requestInterceptor"
				:ref="deepChatRef"
				:initialMessages="initialMessages" />
			<!-- :introMessage="{
					text: `Welcome to Ryobi Tools! Please go ahead with any queries about Ryobi products.`
				}" -->
		</div>
	</div>
</template>
<script>
	import 'deep-chat'
	import { onMounted, ref } from 'vue'
	import axios from 'axios'
	import URLPreview from './components/URLPreview.vue'
	import markdownit from 'markdown-it'
	export default {
		components: { URLPreview },
		name: 'App',
		setup() {
			const md = markdownit({
				html: true,
				linkify: true,
				breaks: true,
				typographer: true
			})
			const url = 'http://localhost:8086'
			const deepChatRef = ref(null)
			var itemType = ''
			let imageURL = ''
			let itemTypeValue = ''
			let imageURLValue = ''
			let showChat = ref(true)
			const styles = {
				backgroundColor: '#f7f7f7',
				borderBottomRightRadius: '8px',
				borderBottomLeftRadius: '8px',
				height: '500px',
				width: '500px'
			}
			const textInputStyles = {
				styles: {
					text: { color: 'black' },
					container: {
						backgroundColor: '#f5f9ff',
						width: '470px',
						height: '38px',
						fontSize: '16px'
					},
					focus: { border: '2px solid #a2a2ff' }
				},
				placeholder: {
					text: 'Insert text here...',
					style: { color: '#4459a4' }
				}
			}
			const submitButtonStyles = {
				submit: {
					container: {
						default: {
							bottom: '1.2em',
							paddingRight: '6px'
						}
					},
					svg: {
						styles: { default: { fontSize: '18px' } }
					}
				},
				loading: {
					container: { height: '50px' }
				},
				text: {
					styles: { height: '10px' }
				}
			}
			const convertToImage  = (img) => {
				let imgHTML = md.render(img)
				const urlRegex = /\bhttps?:\/\/[^\s)]+/gi
				let matches = img.match(urlRegex).slice()
				if (matches && matches.length > 0) {
					imageURL = matches[0].trim()
				} else {
					imageURL = ''
				}
				return imgHTML.replace(/<img([^>]*)>/g, (match, group) => {
					if (!group.includes('width') && !group.includes('height')) {
						return `<img${group} width="200" height="200" style='border-radius:16px'>`
					} else {
						return match
					}
				})
			}
			const optionsList = options => {
				let optionsHTML = ''
				if (typeof options[0] === 'string') {
					options.forEach(element => {
						if (element.includes('https')) {
							if (
								element.includes('![') &&
								!(
									element.includes('jpeg') ||
									element.includes('jpg') ||
									element.includes('png')
								)
							) {
								element = element.replace('!', '')
							}
							if( element.includes('jpeg') ||
								element.includes('jpg') ||
								element.includes('png')
								) {
									optionsHTML += convertToImage(element)
							} else {
								let parts = element.split('-')
								optionsHTML += `<div style="display: flex; justify-content: space-between"> 
																	<button class="deep-chat-button deep-chat-suggestion-button" style="margin-top: 5px; margin-right:2px; margin-left:2px">
																			${parts[0]}
																	</button>${md.render(parts[2])}
																</div>`
							}
						} else if (['Available', 'Not Available'].includes(element)) {
							optionsHTML += `<div>${element}</div>`
						} else {
							optionsHTML += `<button class="deep-chat-button deep-chat-suggestion-button" style="margin-top: 5px; margin-right:2px; margin-left:2px">${element}</button>`
						}
					})
				} else if (typeof options[0] === 'Object') {
					console.log(options[0])
				}
				if(optionsHTML.includes('<img')) {
					return optionsHTML
				}
				return `<div class="deep-chat-temporary-message">${optionsHTML}</div>`
			}
			function extractJSONFromString(inputString) {
				inputString = inputString.replaceAll("'", '"')
				const startIndex = inputString.indexOf('{')
				const endIndex = inputString.lastIndexOf('}') + 1
				let jsonString = inputString.substring(startIndex, endIndex)
				try {
					let jsonData = JSON.parse(jsonString)
					return jsonData
				} catch (error) {
					return 'Failed to resolve response'
				}
			}
			const convertToHtml = data => {
				let tempData = extractJSONFromString(data)
				try {
					var check_flags = null
					if (tempData && tempData.check_flags) {
						check_flags = `<div class="deep-chat-temporary-message">
							<button class="deep-chat-button deep-chat-suggestion-button" style="border: 1px solid green">Yes</button>
							<button class="deep-chat-button deep-chat-suggestion-button" style="border: 1px solid #d80000">No</button>
						</div>`
					}
					if (tempData && !tempData.item_type) {
						return {
							suggestions: '',
							header: tempData.items_list[0],
							check_flags,
							description: tempData.description
						}
					} else {
						itemType = tempData.item_type
						let header = tempData.header
						let optionsHTML = ''
						if (itemType === 'part_availability') {
							header = tempData.items_list[0]
						} if ( itemType === 'distributor_list') {
							optionsHTML = `<ul>
								${tempData.items_list.map(item => `<li>${item}</li>`).join('')}
							</ul>`
						}
						else {
							optionsHTML = optionsList(tempData.items_list)
						}
						return {
							header,
							suggestions: optionsHTML,
							check_flags,
							description: tempData.description
						}
					}
				} catch (e) {
					tempData = data
				}
				return {
					header: '',
					suggestions: tempData,
					check_flags,
					description: tempData.description
				}
			}

			const requestInterceptor = async req => {
				return req
			}
			const handler = async (body, signals) => {
				signals.onOpen()
				signals.newUserMessage.listener = async (body) => {
					let user_query = body.messages[0].text
					if (itemType === 'product_image_url') {
						itemTypeValue = itemType
						if(['yes', 'Yes'].includes(body.messages[0].text))  {
							user_query = imageURL
						} else if(['No','no'].includes(body.messages[0].text)) {
							user_query = 'Incorrect Image. Please send all categories'
						}
					} else if (itemType === 'distributor_list') {
						user_query = ''
					}
					try {
						signals.onResponse({html: '<div class="loading-message-text custom-loading"><div class="dots-flashing"></div></div>'})
						let res = await axios.post(
							url + '/get_chat_response',
							{
								user_query,
								type: itemTypeValue
							},
							{ withCredentials: true }
						)
						itemTypeValue = ''
						itemType = ''
						let { suggestions, header, check_flags } = convertToHtml(
							res.data.data
						)
						if(header) {
							signals.onResponse({ html: `<div>${header}</div>`, role: 'ai', overwrite: true})
						}
						if (suggestions) {
							signals.onResponse({ html: suggestions, role: 'ai', overwrite: !header})
						}
						if(check_flags) {
							signals.onResponse({html: check_flags, role: 'ai'})
						}
					} catch (e) {
						signals.onResponse({ error: 'Error' })
					}
				}
			}
			let initialMessages = ref([])
			let htmlClassUtilities = {
				'deep-chat-suggestion-button': {
					events: {
						click: data => {
							itemTypeValue = itemType
							imageURLValue = imageURL
						}
					}
				},
				'deep-chat-button': {
					events: {
						click: data => console.log(data)
					}
				}
			}
			let messageStyles = {
				'custom-loading': {
					styles: {
						default: {
							padding: '0.18em 0.1em 0.1em 0.6em',
						},
					},
				},
				loading: {
					bubble: { padding: '20px' }
				},
				default: {
					user: {
						bubble: {
							backgroundColor: '#e1e723',
							padding: '10px',
							color: 'black',
							borderBottomLeftRadius: '16px',
							borderBottomRightRadius: '16px',
							borderTopLeftRadius: '16px',
							borderTopRightRadius: '0px'
						}
					},
					ai: {
						bubble: {
							maxWidth: '70%',
							padding: '10px',
							borderBottomLeftRadius: '16px',
							borderBottomRightRadius: '16px',
							borderTopRightRadius: '16px',
							borderTopLeftRadius: '0px'
						}
					}
				}
			}
			let header_map = {
				product_category:
					'Please select from one of the following product categories:',
				model_number: 'Please select the model number:',
				part_name_part_location_number: 'Please select the part name:',
				product_image_url: 'Is the following image of the product correct?',
				part_url: 'Is the following image of the part correct?',
				part_availability: 'Please select ',
				distributor_list: 'Please check the below distributor list'
			}
			const getChatHistory = () => {
				axios
					.get(url + '/get_chat_history', {
						withCredentials: true
					})
					.then(res => {
						if (res.data.chat_history.length > 0) {
							initialMessages.value = res.data.chat_history.map(msg => {
								// let content = convertToHtml(msg.content)
								let content = ''
								try {
									content = JSON.parse(msg.content)
								} catch (e) {
									content = msg.content
								}
								console.log(content)
								return {
									role: msg.role === 'assistant' ? 'ai' : msg.role,
									text:
										(content.item_type && header_map[content.item_type]) || content
								}
							})
						}
					})
			}
			onMounted(async () => {
				let session_id =
					document.cookie
						.split(';')
						.filter(cookie => cookie.match('session_id')).length > 0
						? document.cookie
								.split(';')
								.filter(cookie => cookie.match('session_id'))[0]
								.split('=')[1]
						: ''
				if (!session_id) await axios.get(url, { withCredentials: true })
				getChatHistory()
			})
			return {
				url,
				styles,
				initialMessages,
				textInputStyles,
				handler,
				messageStyles,
				showChat,
				submitButtonStyles,
				getChatHistory,
				requestInterceptor,
				htmlClassUtilities,
				itemType,
				imageURL,
				itemTypeValue,
				imageURLValue,
				deepChatRef
			}
		}
	}
</script>

<style>
	.header {
		background-color: #e1e723;
		color: #000;
		display: flex;
		justify-content: space-between;
		align-items: center;
		padding: 10px 20px;
		border-radius: 4px;
	}

	.logo {
		display: flex;
		align-items: center;
	}

	.logo img {
		height: 40px;
		margin-right: 10px;
	}

	.menu {
		display: flex;
	}

	.menu li {
		list-style-type: none;
		margin-right: 20px;
	}

	.menu li:last-child {
		margin-right: 0;
	}

	.menu li a {
		color: #000;
		text-decoration: none;
	}
	.container {
		position: fixed;
		right: 0;
		bottom: 0;
		margin-right: 20px;
		margin-bottom: 20px;
	}
	.chat-header {
		height: 30px;
		width: 482px;
		padding: 10px;
		color: #000;
		background: #e1e723;
		border-top-left-radius: 8px;
		border-top-right-radius: 8px;
		display: flex;
		justify-content: space-between;
		align-items: center;
	}
	.inside-right {
		bottom: 1.2em !important;
	}
	:root {
		--message-dots-color: #848484;
		--message-dots-color-fade: #55555533;
	}
</style>
