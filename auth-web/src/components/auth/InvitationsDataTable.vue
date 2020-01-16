<template>
  <v-container class="view-container">
  <v-data-table
    class="user-list"
    :headers="headerInvitations"
    :items="indexedInvitations"
    :items-per-page="5"
    :calculate-widths="true"
    :hide-default-footer="indexedInvitations.length <= 5"
    :no-data-text="$t('noPendingInvitesLabel')"
  >
    <template v-slot:item.recipientEmail="{ item }" >
      <span :data-test="getIndexedTag('invitation-email', item.index)">{{ item.recipientEmail }}</span>
    </template>
    <template v-slot:item.sentDate="{ item }">
      <span :data-test="getIndexedTag('invitation-sent', item.index)">{{ formatDate (item.sentDate) }}</span>
    </template>
    <template v-slot:item.expiresOn="{ item }">
      <span :data-test="getIndexedTag('invitation-expires', item.index)">{{ formatDate (item.expiresOn) }}</span>
    </template>
    <template v-slot:item.action="{ item }">
      <v-btn :data-test="getIndexedTag('resend-button', item.index)" small color="primary" class="mr-2" @click="resend(item)">Resend</v-btn>
      <v-btn :data-test="getIndexedTag('remove-button', item.index)" depressed small @click="showConfirmRemoveInviteModal(item)">Remove</v-btn>
    </template>
  </v-data-table>

    <!-- Alert Dialog (Success) -->
    <ModalDialog
            ref="successDialog"
            :title="successTitle"
            :text="successText"
            dialog-class="notify-dialog"
            max-width="640"
    ></ModalDialog>
    <ModalDialog
            ref="confirmActionDialog"
            :title="confirmActionTitle"
            :text="confirmActionText"
            dialog-class="notify-dialog"
            max-width="640"
    >
      <template v-slot:icon>
        <v-icon large color="error">mdi-alert-circle-outline</v-icon>
      </template>
      <template v-slot:actions>
        <v-btn large color="primary" @click="confirmHandler()">{{ primaryActionText }}</v-btn>
        <v-btn large color="default" @click="cancel()">{{ secondaryActionText }}</v-btn>
      </template>
    </ModalDialog>

  </v-container>
</template>

<script lang="ts">
import { Component, Emit, Vue } from 'vue-property-decorator'
import { mapActions, mapState } from 'vuex'
import { Invitation } from '@/models/Invitation'
import ModalDialog from '@/components/auth/ModalDialog.vue'
import moment from 'moment'

@Component({
  components: {
    ModalDialog
  },
  computed: {
    ...mapState('org', ['pendingOrgInvitations', 'sentInvitations'])
  },
  methods: {
    ...mapActions('org', [
      'resendInvitation',
      'deleteInvitation',
      'updateMember',
      'approveMember',
      'leaveTeam',
      'syncPendingOrgInvitations'
    ])
  }
})
export default class InvitationsDataTable extends Vue {
  private readonly pendingOrgInvitations!: Invitation[]
  private readonly syncPendingOrgInvitations!: () => Invitation[]
  private readonly resendInvitation!: (invitation: Invitation) => void
  private readonly sentInvitations!: Invitation[]
  private invitationToBeRemoved: Invitation
  private successTitle: string = ''
  private successText: string = ''
  private confirmActionTitle: string = ''
  private confirmActionText: string = ''
  private confirmHandler: () => void = undefined
  private primaryActionText: string = ''
  private readonly deleteInvitation!: (invitationId: number) => void
  private secondaryActionText = 'No'

  $refs: {
    successDialog: ModalDialog,
    confirmActionDialog: ModalDialog
  }

  private readonly headerInvitations = [
    {
      text: 'Email',
      align: 'left',
      sortable: true,
      value: 'recipientEmail'
    },
    {
      text: 'Invitation Sent',
      align: 'left',
      sortable: true,
      value: 'sentDate'
    },
    {
      text: 'Expires',
      align: 'left',
      sortable: true,
      value: 'expiresOn'
    },
    {
      text: 'Actions',
      align: 'left',
      value: 'action',
      sortable: false,
      width: '193'
    }
  ]

  private async mounted () {
    await this.syncPendingOrgInvitations()
  }
  private cancel () {
    this.$refs.confirmActionDialog.close()
  }
  private async resend (invitation: Invitation) {
    await this.resendInvitation(invitation)
    this.showSuccessModal()
  }

  private formatDate (date: Date) {
    return moment(date).format('DD MMM, YYYY')
  }

  private getIndexedTag (tag, index): string {
    return `${tag}-${index}`
  }

  private get indexedInvitations () {
    return this.pendingOrgInvitations.map((item, index) => ({
      index,
      ...item
    }))
  }

  private showConfirmRemoveInviteModal (invitation: Invitation) {
    this.confirmActionTitle = this.$t('confirmRemoveInviteTitle').toString()
    this.confirmActionText = `Are you sure wish to remove the invite to ${invitation.recipientEmail}?`
    this.invitationToBeRemoved = invitation
    this.confirmHandler = this.removeInvite
    this.primaryActionText = 'Remove'
    this.$refs.confirmActionDialog.open()
  }

  private showSuccessModal () {
    this.successTitle = `Invited ${this.sentInvitations.length} Team Members`
    this.successText = 'Your team invitations have been sent successfully.'
    this.$refs.successDialog.open()
  }
  private async removeInvite () {
    await this.deleteInvitation(this.invitationToBeRemoved.id)
    this.$refs.confirmActionDialog.close()
  }
}
</script>
