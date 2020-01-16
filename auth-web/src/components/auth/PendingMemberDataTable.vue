<template>
  <v-container class="view-container">
    <v-data-table
            class="user-list"
            :headers="headerPendingMembers"
            :items="indexedPendingMembers"
            :items-per-page="5"
            :calculate-widths="true"
            :hide-default-footer="indexedPendingMembers.length <= 5"
            :custom-sort="customSortPending"
            :no-data-text="$t('noPendingApprovalLabel')"
    >
      <template v-slot:loading>
        Loading...
      </template>
      <template v-slot:item.name="{ item }">
        <v-list-item-title class="user-name" :data-test="getIndexedTag('pending-user-name', item.index)">{{
          item.user.firstname }} {{ item.user.lastname }}
        </v-list-item-title>
        <v-list-item-subtitle :data-test="getIndexedTag('pending-email', item.index)"
                              v-if="item.user.contacts && item.user.contacts.length > 0">{{ item.user.contacts[0].email
          }}
        </v-list-item-subtitle>
      </template>
      <template v-slot:item.action="{ item }">
        <v-btn :data-test="getIndexedTag('approve-button', item.index)" small color="primary" class="mr-2"
               @click="confirmApproveMember(item)">Approve
        </v-btn>
        <v-btn :data-test="getIndexedTag('deny-button', item.index)" depressed small @click="confirmDenyMember(item)">
          Deny
        </v-btn>
      </template>
    </v-data-table>

    <!-- Confirm Action Dialog -->
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
import { Member, MembershipStatus, Organization, UpdateMemberPayload } from '@/models/Organization'
import { mapActions, mapState } from 'vuex'
import ModalDialog from '@/components/auth/ModalDialog.vue'
import moment from 'moment'

  @Component({
    components: {
      ModalDialog
    },
    computed: {
      ...mapState('org', ['pendingOrgMembers'])
    },
    methods: {
      ...mapActions('org', [
        'resendInvitation',
        'deleteInvitation',
        'updateMember',
        'approveMember',
        'leaveTeam',
        'syncPendingOrgMembers'
      ])
    }
  })
export default class PendingMemberDataTable extends Vue {
    private readonly pendingOrgMembers!: Member[]
    private readonly syncPendingOrgMembers!: () => Member[]
    private memberToBeApproved: Member
    private memberToBeRemoved: Member

    private readonly headerPendingMembers = [
      {
        text: 'Team Member',
        align: 'left',
        sortable: true,
        value: 'name'
      },
      {
        text: 'Actions',
        align: 'left',
        value: 'action',
        sortable: false,
        width: '195'
      }
    ]

    private confirmActionTitle: string = ''
    private confirmActionText: string = ''
    private primaryActionText: string = ''
    private secondaryActionText = 'No'
    private readonly updateMember!: (updateMemberPayload: UpdateMemberPayload) => void
    private confirmHandler: () => void = undefined

    $refs: {
      successDialog: ModalDialog
      errorDialog: ModalDialog
      inviteUsersDialog: ModalDialog
      confirmActionDialog: ModalDialog
      confirmActionDialogWithQuestion: ModalDialog
    }

    private async mounted () {
      await this.syncPendingOrgMembers()
    }

    private getIndexedTag (tag, index): string {
      return `${tag}-${index}`
    }

    private get indexedPendingMembers () {
      return this.pendingOrgMembers.map((item, index) => ({
        index,
        ...item
      }))
    }

    private customSortPending (items, index, isDescending) {
      const isDesc = isDescending.length > 0 && isDescending[0]
      items.sort((a, b) => {
        if (isDesc) {
          return a.user.firstname < b.user.firstname ? -1 : 1
        } else {
          return b.user.firstname < a.user.firstname ? -1 : 1
        }
      })
      return items
    }

    @Emit()
    private confirmApproveMember (member: Member) {
      this.confirmActionTitle = this.$t('confirmApproveMemberTitle').toString()
      this.confirmActionText = `Are you sure you wish to approve membership for ${member.user.firstname}?`
      this.memberToBeApproved = member
      this.confirmHandler = this.approve
      this.primaryActionText = 'Approve'
      this.$refs.confirmActionDialog.open()
    }

    private async approve () {
      await this.updateMember({
        memberId: this.memberToBeApproved.id,
        status: MembershipStatus.Active
      })
      this.$refs.confirmActionDialog.close()
    }

    @Emit()
    private confirmDenyMember (member: Member) {
      if (member.membershipStatus === MembershipStatus.Pending) {
        this.confirmActionTitle = this.$t('confirmDenyMemberTitle').toString()
        this.confirmActionText = `Are you sure you want to deny membership to ${member.user.firstname}?`
        this.confirmHandler = this.deny
        this.primaryActionText = 'Deny'
      }
      this.memberToBeRemoved = member
      this.$refs.confirmActionDialog.open()
    }

    private async deny () {
      await this.updateMember({
        memberId: this.memberToBeRemoved.id,
        status: MembershipStatus.Rejected
      })
      this.$refs.confirmActionDialog.close()
    }
    private cancel () {
      this.$refs.confirmActionDialog.close()
    }
}
</script>
