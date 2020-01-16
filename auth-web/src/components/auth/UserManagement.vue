<template>
  <v-container class="view-container">
    <header class="view-header">
      <h1>Manage Team</h1>
      <div class="view-header__actions">
        <v-btn v-if="canInvite()" large color="primary" @click="showInviteUsersModal()" data-test="invite-people-button">
          <v-icon>mdi-plus</v-icon>
          <span>Invite People</span>
        </v-btn>
      </div>
    </header>

    <!-- Tab Navigation -->
    <v-tabs class="mb-7" v-model="tab" background-color="transparent">
      <v-tab data-test="active-tab" to="/main/team/active">Active</v-tab>
      <v-tab data-test="pending-approval-tab" to="/main/team/pending" v-show="canInvite()">Pending Approval</v-tab>
      <v-tab data-test="invitations-tab" to="/main/team/invitations" v-show="canInvite()">Invitations</v-tab>
    </v-tabs>

    <router-view></router-view>

    <!-- Invite Users Dialog -->
    <ModalDialog
      ref="inviteUsersDialog"
      :show-icon="false"
      :show-actions="false"
      :fullscreen-on-mobile="$vuetify.breakpoint.xsOnly || $vuetify.breakpoint.smOnly || $vuetify.breakpoint.mdOnly"
      :is-persistent="true"
      :is-scrollable="true"
      max-width="640"
    >
      <template v-slot:title>
        <span>Invite Team Members</span>
      </template>
      <template v-slot:text>
        <InviteUsersForm @invites-complete="showSuccessModal()" @cancel="cancelInviteUsersModal()" />
      </template>
    </ModalDialog>

    <!-- Alert Dialog (Success) -->
    <ModalDialog
      ref="successDialog"
      :title="successTitle"
      :text="successText"
      dialog-class="notify-dialog"
      max-width="640"
    ></ModalDialog>

  </v-container>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator'
import { Member, MembershipStatus, MembershipType, Organization } from '@/models/Organization'
import { mapActions, mapGetters, mapState } from 'vuex'
import { Business } from '@/models/business'
import { Invitation } from '@/models/Invitation'
import InvitationsDataTable from '@/components/auth/InvitationsDataTable.vue'
import InviteUsersForm from '@/components/auth/InviteUsersForm.vue'
import MemberDataTable from '@/components/auth/MemberDataTable.vue'
import ModalDialog from '@/components/auth/ModalDialog.vue'
import OrgModule from '@/store/modules/org'
import PendingMemberDataTable from '@/components/auth/PendingMemberDataTable.vue'
import { getModule } from 'vuex-module-decorators'

  @Component({
    components: {
      MemberDataTable,
      InvitationsDataTable,
      PendingMemberDataTable,
      InviteUsersForm,
      ModalDialog
    },
    computed: {
      ...mapState('org', [
        'resending',
        'sentInvitations',
        'myOrgMembership'
      ]),
      ...mapState('business', ['currentBusiness'])
    },
    methods: {
      ...mapActions('org', [
        'syncOrganizations',
        'syncMyOrgMembership'
      ])
    }
  })
export default class UserManagement extends Vue {
  private orgStore = getModule(OrgModule, this.$store)
  private successTitle: string = ''
  private successText: string = ''
  private tab = null
  private isLoading = true

  private readonly currentBusiness!: Business
  private readonly resending!: boolean
  private readonly sentInvitations!: Invitation[]
  private readonly myOrgMembership!: Member
    private readonly syncMyOrgMembership!:() => Member

  private readonly syncOrganizations!: () => Promise<Organization[]>

  $refs: {
    successDialog: ModalDialog
    inviteUsersDialog: ModalDialog
    errorDialog: ModalDialog
  }

  private async mounted () {
    await this.syncMyOrgMembership()
    this.isLoading = false
  }

  private canInvite (): boolean {
    return this.myOrgMembership &&
            this.myOrgMembership.membershipStatus === MembershipStatus.Active &&
            (this.myOrgMembership.membershipTypeCode === MembershipType.Owner ||
             this.myOrgMembership.membershipTypeCode === MembershipType.Admin)
  }

  private showInviteUsersModal () {
    this.$refs.inviteUsersDialog.open()
  }

  private cancelInviteUsersModal () {
    this.$refs.inviteUsersDialog.close()
  }

  private showSuccessModal () {
    this.$refs.inviteUsersDialog.close()
    this.successTitle = `Invited ${this.sentInvitations.length} Team Members`
    this.successText = 'Your team invitations have been sent successfully.'
    this.$refs.successDialog.open()
  }

  private close () {
    this.$refs.errorDialog.close()
  }
}
</script>

<style lang="scss" scoped>
  .view-header {
    display: flex;
    flex-direction: row;
    justify-content: space-between;
    padding-top: 1.5rem;
    padding-bottom: 1rem;

    h1 {
      margin-bottom: 0;
    }

    .v-btn {
      font-weight: 700;
    }
  }

  ::v-deep {
    .v-data-table td {
      padding-top: 1rem;
      padding-bottom: 1rem;
      height: auto;
      vertical-align: top;
    }

    .v-list-item__title {
      display: block;
      font-weight: 700;
    }
  }

  .notify-checkbox {
    justify-content: center;

    ::v-deep {
      .v-input__slot {
        margin-bottom: 0 !important;
      }
    }
  }
</style>
