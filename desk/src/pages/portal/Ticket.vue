<template>
	<div v-if="ticket" class="max-w-4xl mx-auto">
		<div 
			:style="{ height: viewportWidth > 768 ? 'calc(100vh - 14rem)' : null }"
		>
			<div class="flex justify-between mb-4">
				<div class="flex items-center space-x-2 text-lg">
					<a href="/support/kb">Home</a>
					<FeatherIcon name="chevron-right" class="h-3 w-3"/>
					<router-link :to="{name: 'ProtalTickets'}">Tickets</router-link>
					<FeatherIcon name="chevron-right" class="h-3 w-3"/>
					<div class="text-gray-400">{{ ticket.name }}</div>
				</div>
				<div>
					<Button v-if="['Open', 'Replied'].includes(ticket.status)" @click="closeTicket()" class="bg-gray-100 text-red-500">Close</Button>
				</div>
			</div>
			<div class="flex items-center pb-2 justify-between">
				<div class="text-5xl font-bold">
					{{ ticket.subject }}
				</div>
			</div>
			<div class="grow flex flex-col h-full space-y-2">
				<div class="overflow-auto grow">
					<div v-if="['Closed', 'Resolved'].includes(ticket.status)" class="w-full bg-gray-100 rounded-[8px] py-[10px] px-[15px] flex flex-row items-center justify-between mb-[12px]">
						<div class="flex flex-row items-center space-x-[9px]">
							<CustomIcons name="circle-check" class="w-[18px] h-[18px]" />
							<div class="text-base">This ticket has been closed.</div>
						</div>
						<div class="text-[#096CC3] text-[12px] font-medium cursor-pointer hover:text-blue-500" @click="reopenTicket()">Reopen ticket</div>
					</div>
					<CustomerSatisfactionFeedback :ticket="ticket" v-if="['Closed', 'Resolved'].includes(ticket.status)" />
					<Conversations :ticketId="ticket.name" :scrollToBottom="scrollConversationsToBottom"/>
				</div>
				<div v-if="!editing && ['Open', 'Replied'].includes(ticket.status)" class="mt-5 ml-9">
					<Button @click="startEditing()" appearance="primary">Reply</Button>
				</div>
				<div
					class="flex flex-col" 
				>
					<div class="grow">
						<div class="flex">
							<Avatar v-if="editing" :label="user" :imageURL="user.profile_image" size="md" />
							<div class="w-full ml-2 pt-1">
								<div class="flex flex-col space-y-2">
									<div v-if="editing" class="text-lg">{{ user.doc.full_name }}</div>
									<div class="grow" v-if="!(['Closed', 'Resolved'].includes(ticket.status))">
										<CustomTextEditor 
											ref="replyEditor"
											:show="editing"
											v-on:keydown="handleShortcuts($event)"
											@click="$refs.replyEditor.focusEditor()"
											:content="content" 
											@change="(val) => { content = val }"
											placeholder="Type a response"
											editorClasses="w-full min-h-[180px] max-h-[300px] " class="border border-gray-300 rounded-[8px] p-[12px]"
										>
											<template #bottom-section="{ editor }">
												<div>
													<div v-if="attachments.length" class="max-h-[100px] overflow-y-scroll rounded flex flex-col">
														<ul class="flex flex-wrap gap-2 py-2">
															<li
																class="flex items-center p-1 space-x-2 bg-gray-100 border-gray-400 border shadow rounded"
																v-for="file in attachments"
																:key="file"
																:title="file.name"
															>
																<div class="flex flex-row items-center space-x-1">
																	<FeatherIcon name="file-text" class="h-[15px] stroke-gray-600"/>
																	<span class="text-[12px] text-gray-700 font-normal ml-2 max-w-[100px] truncate">
																		{{ file.file_name }}
																	</span>
																</div>
																<button class="grid w-4 h-4 text-gray-700 rounded hover:bg-gray-300 place-items-center" @click="() => { attachments = attachments.filter(x => x.name != file.name) }">
																	<FeatherIcon class="w-3" name="x" />
																</button>
															</li>
														</ul>
													</div>
													<div v-if="showTextFormattingMenu">
														<div class="flex flex-row items-center space-x-1.5 p-1.5 rounded shadow w-fit">
															<div v-for="item in [
																'bold', 'italic', '|',
																'quote', 'code', '|',
																'numbered-list', 'bullet-list', 'left-align', 'center-align', 'right-align'
															]" :key="item">
																<TextEditorMenuItem :item="item" :editor="editor" :attachments="attachments" />
															</div>
														</div>
													</div>
													<div class="pt-2 select-none flex flex-row items-center space-x-2">
														<Button 
															:loading="$resources.submitConversation.loading" 
															@click="submitConversation()" 
															appearance="primary" 
															:disabled="sendingDissabled"
														>
															Send
														</Button>
														<Button @click="cancelEditing">Cancel</Button>
														<div class="flex flex-row items-center space-x-2">
															<CustomIcons :class="showTextFormattingMenu ? 'bg-gray-200' : ''" name="text-formatting" class="h-7 w-7 rounded p-1" role="button" @click="() => { showTextFormattingMenu = !showTextFormattingMenu}"/>
															<FileUploader @success="(file) => attachments.push(file)">
																<template v-slot="{ progress, uploading, openFileSelector }">
																	<FeatherIcon name="paperclip" class="h-[17px]" @click="openFileSelector" role="button" :disabled="uploading" />
																</template>
															</FileUploader>
															<CustomIcons :class="editor.isActive('link') ? 'bg-gray-100' : ''" name="link-url" class="h-7 w-7 rounded p-1" role="button" @click="$refs.replyEditor.insertLink"/>
														</div>
														<div class="grow flex flex-row-reverse">
															<FeatherIcon name="trash-2" role="button" class="h-4 w-4" @click="() => {
																content = ''
																attachments = []
															}" />
														</div>
													</div>
												</div>
											</template>
										</CustomTextEditor>
									</div>
								</div>
							</div>
						</div>
					</div>
				</div>
			</div>
		</div>
	</div>
</template>

<script>
import { inject, ref } from "vue"
import Conversations from "@/components/portal/ticket/Conversations.vue"
import CustomTextEditor from "@/components/global/CustomTextEditor.vue"
import ActionPanel from "@/components/portal/ticket/ActionPanel.vue"
import { FeatherIcon, FileUploader, TextEditor, Avatar } from 'frappe-ui'
import CustomIcons from "@/components/desk/global/CustomIcons.vue";
import CustomerSatisfactionFeedback from "@/components/portal/ticket/CustomerSatisfactionFeedback.vue";
import TextEditorMenuItem from '@/components/global/TextEditorMenuItem.vue';

export default {
    name: "Tickets",
    props: ["ticketId"],
	components: {
		Conversations,
		CustomTextEditor,
		ActionPanel,
		FeatherIcon,
		TextEditor,
		FileUploader,
		CustomIcons,
		CustomerSatisfactionFeedback,
		TextEditorMenuItem,
		Avatar
	},
	data() {
		return {
			editing: false,
			scrollConversationsToBottom: false,
			content: "",
		}
	},
    setup() {
		const user = inject('user')
		const editor = ref(null)
		const viewportWidth = inject("viewportWidth")
        const tickets = inject("tickets")
        const ticketController = inject("ticketController")
		const attachments = ref([])
		const tempTextEditorData = ref({})
		const showTextFormattingMenu = ref(true)
        
		return { user, editor, viewportWidth, tickets, ticketController, attachments, tempTextEditorData, showTextFormattingMenu }
    },
    computed: {
        ticket() {
			if (this.tickets) {
				return this.tickets[this.ticketId] || null
			} else {
				return null
			}
        },
		sendingDissabled() {
			let content = this.content.trim()
			content = content.replaceAll('<p></p>', '')
			content = content.replaceAll(' ', '')
			return (content == "" || content == "<p><br></p>" || content == '<p></p>') && this.attachments.length == 0
		}
    },
	resources: {
		submitConversation() {
			return {
				method: 'frappedesk.api.ticket.submit_conversation_via_contact',
				onSuccess: () => {
					this.tempTextEditorData = {}
					this.editing = false
				},
				onError: () => {
					this.content = this.tempTextEditorData.content
					this.attachments = this.tempTextEditorData.attachments
				}
			}
		}
	},
	methods: {
		handleShortcuts(e) {
			if ((e.metaKey || e.ctrlKey) && e.keyCode == 13) {
				if (!this.sendingDissabled) {
					this.submitConversation()
				}
			}
			else if ((e.metaKey || e.ctrlKey) && e.keyCode == 75) {
				this.$refs.replyEditor.insertLink()
			}
			else if (e.keyCode === 27) {
				this.cancelEditing()
			}
		},
		startEditing() {
			this.editing = true
			this.delayedConversationScroll()
			this.$refs.replyEditor.focusEditor()
		},
		cancelEditing() {
			this.editing = false
		},
		delayedConversationScroll() {
			function delay(time) {
				return new Promise(resolve => setTimeout(resolve, time));
			}

			delay(400).then(() => this.scrollConversationsToBottom = true)
			delay(1000).then(() => this.scrollConversationsToBottom = false)
		},
		submitConversation() {
			this.tempTextEditorData.content = this.content
			this.tempTextEditorData.attachments = this.attachments

			this.$resources.submitConversation.submit({
				ticket_id: this.ticketId,
				message: this.content,
				attachments: this.attachments.map(x => x.name)
			})

			this.attachments = []
			this.content = ""
		},
		closeTicket() {
			this.ticketController.set(this.ticketId, 'status', 'Closed')
		},
		reopenTicket() {
			this.ticketController.set(this.ticketId, 'status', 'Open')
		}
	}
}
</script>