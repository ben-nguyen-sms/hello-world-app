<template>
    <v-app v-if="loaded">
        <v-container fluid class="pa-0">
            <v-main :class="{'mt-16 mb-16': $store.state.isNotImpersonationMode, 'mb-16': !$store.state.isNotImpersonationMode}">
                <TheToolbar v-if="$store.state.isNotImpersonationMode" />
                <DashboardHero :model="dashboardBanner" />
                <div :class="isMobile || isTablet ? 'mobile-container' : 'container'">
                    <div class="div-content content-justify-space-between">
                        <Tabs :model="model" class="border-bottom mb-2" :tabs="tabs">
                            <Tab v-for="(tab, index) in tabs"
                                 :index="index"
                                 :showDividers="true"
                                 :key="tab.id"
                                 :tab="tab"
                                 :hasBorder="tabs[tabs.length - 1].id !== tab.id"
                                 :isDisabled="isPasswordExpired && tab.id != 'changePassword'" />
                        </Tabs>
                        <div id="bottomLine" class="bottomLine"></div>
                        <div class="return-to-dashboard">
                            <router-link to="/dashboard" class="link">
                                <v-icon> mdi-chevron-left </v-icon>
                                RETURN TO DASHBOARD
                            </router-link>
                        </div>
                    </div>
                    <v-tabs-items v-model="model.tab">
                        <v-tab-item :value="0">
                            <v-container fluid class="pa-0">
                                <v-row role="region" aria-labelledby="region-message">
                                    <v-col class="col">
                                        <div id="region-message">
                                            <p v-if="changePasswordForm.pwDisallowReuse != 0" class="mt-8" id="pw-reuse">{{iterationRequirementMessage}}</p>
                                            <span class="span-visually-hidden">Change Password</span>
                                        </div>
                                        <ChangePassword :changePasswordForm.sync="changePasswordForm"
                                                        @changePasswordButtonAction="changePasswordButtonAction" />
                                        <TheMessageDialog :show-dialog="showAlertDialog"
                                                          :message="verifyCurrentPasswordADA"
                                                          :title="titleAlert"
                                                          @clickOK="processOKDialog" />
                                        <p v-if="passwordReuseError" class="error--text reuse-error">
                                            {{iterationErrorMessage}}
                                        </p>
                                        <template>
                                            <div id="divIterationErrorMessageADA" class="span-visually-hidden" :aria-live='isChromeBrowser ? "assertive" : "polite"'>
                                                {{this.iterationErrorMessageADA}}
                                            </div>
                                        </template>
                                    </v-col>
                                </v-row>
                            </v-container>
                        </v-tab-item>
                        <v-tab-item :value="1">
                            <v-container>
                                <v-row>
                                    <v-col class="col-xl-6 col-lg-7 col-md-9 col-sm-10 col-xs-12">
                                        <ManageProfileForm ref="manageProfileForm"
                                                           v-bind:syncModel.sync="manageProfileModel"
                                                           :options="manageProfileOptions" />

                                        <BackContinueButtons :key="nextButtonKey"
                                                             :backButton="false"
                                                             :continueButton="true"
                                                             :customContinueText="continueText"
                                                             :ariaDescribedby="manageProfileModel.tcpaWarningText != '' ? 'tcpaWarningText' : ''"
                                                             @nextButtonAction="nextButtonAction" />
                                    </v-col>
                                </v-row>
                            </v-container>
                        </v-tab-item>
                    </v-tabs-items>
                </div>
            </v-main>
            <Footer v-show="loaded" />
            <SuccessFailSnackbar v-if="snackBarShow"
                                 :showSnackbar="snackBarShow"
                                 :snackbarText="snackBarMessage"
                                 :activeClass="snackBarClass"
                                 @close="closeSnackBar">
            </SuccessFailSnackbar>
        </v-container>
    </v-app>
</template>

<script lang="ts">
    import { Component, Mixins, Watch } from 'vue-property-decorator'
    import { ChangePassword, ManageProfileForm } from 'velocity.akouba.common.vue'
    import ManageProfile from 'velocity.akouba.common.vue/src/ts/models/ManageProfile'
    import ManageProfileOptions from 'velocity.akouba.common.vue/src/ts/models/ManageProfileOptions'
    import CommonAuthUtilMixin from 'velocity.akouba.common.vue/src/ts/mixins/CommonAuthUtilMixin'
    import MobileMixin from 'velocity.akouba.common.vue/src/ts/mixins/MobileMixin'
    import SnackbarMixin from 'velocity.akouba.common.vue/src/ts/mixins/SnackbarMixin'
    import TheToolbar from '@/components/Shared/TheToolbar.vue'
    import Footer from '@/components/Shared/Footer.vue'
    import DashboardHero from '@/components/Dashboard/DashboardHero.vue'
    import Tabs from '@/components/Shared/Tabs.vue'
    import Tab from '@/components/Shared/Tab.vue'
    import TabModel from '@/ts/models/TabModel'
    import routeConstants from '@/router/RouteConstants'
    import BackContinueButtons from '@/components/Shared/BackContinueButtons.vue'
    import axiosInstance from '@/plugins/axios-instance'
    import SuccessFailSnackbar from '@/components/Shared/SuccessFailSnackbar.vue'
    import TitleMixin from '@/ts/mixins/TitleMixin'
    import { MfaForm } from '@/ts/enums/MfaForm'
    import TheMessageDialog from '@/components/Shared/TheMessageDialog.vue'
    import navigatorMixin from '@/mixins/navigatorMixin'
    import axiosInstanceCommon from '@/plugins/axios-instance-common'
    import ChangePasswordForm from 'velocity.akouba.common.vue/src/ts/models/ChangePasswordForm'

    class DashboardBanner {
        banner: string = ''
        bannerAlt: string = ''
        header: string = ''
        subHeader: string = ''
    }

    @Component({
        name: "ManageAccount",
        components: {
            TheToolbar,
            DashboardHero,
            Tabs,
            Tab,
            Footer,
            ChangePassword,
            TheMessageDialog,
            ManageProfileForm,
            BackContinueButtons,
            SuccessFailSnackbar
        },
        mixins: [TitleMixin, MobileMixin, SnackbarMixin, navigatorMixin, CommonAuthUtilMixin],
    })
    export default class ManageAccount extends Mixins(MobileMixin, SnackbarMixin, TitleMixin, CommonAuthUtilMixin) {
        private loaded: boolean = false
        private isTcpaInitialValue: boolean | null = null
        private mobilePhoneNumberInitialValue: string | null = null
        private tcpaWarningText: string = ''
        private iterationErrorMessageADA: string = ''
        private verifyCurrentPasswordADA: string = ''
        private titleAlert: string = ''
        private showAlertDialog: boolean = false
        private isChangedSuccess: boolean = false
        private isAccountLocked: boolean = false
        private isChromeBrowser: boolean = false

        public continueText: string = 'Save Changes'
        public nextButtonKey: number = 0
        public activeTab: string = "changePassword"
        public manageProfileModel: ManageProfile = new ManageProfile()
        public manageProfileOptions: ManageProfileOptions = new ManageProfileOptions()

        private changePasswordForm: ChangePasswordForm = new ChangePasswordForm()
        public dashboardBanner: DashboardBanner = new DashboardBanner()

        public model: any = {
            tab: this.activeTab
        }

        public tabs: Array<TabModel> = [
            {
                id: 'changePassword',
                name: 'Change Password',
                route: ''
            },
            {
                id: 'manageProfile',
                name: 'Manage Profile',
                route: ''
            },
        ]

        public get iterationRequirementMessage(): string {
            return this.$store.state.Account.changePassword.iterationRequirementMessage
        }

        public get iterationErrorMessage(): string {
            return this.$store.state.Account.changePassword.iterationErrorMessage
        }

        public get passwordReuseError(): boolean {
            let state = this.$store.state.Account.passwordReuseError
            return state?.error === true
        }

        public get isPasswordExpired(): boolean {
            return sessionStorage.isPasswordExpired == "true"
        }

        public async nextButtonAction(): Promise<void> {
            // @ts-ignore
            let vuelidateManageProfile: any = this.$refs.manageProfileForm.$v

            // Were changes detected
            if (vuelidateManageProfile.$anyDirty || this.isTcpaInitialValue != this.manageProfileModel.isTcpa) {
                if (!vuelidateManageProfile.$invalid) {
                    await this.saveChanges();

                    vuelidateManageProfile.$reset()
                } else {
                    vuelidateManageProfile.$touch()
                    let invalidField = Object.keys(vuelidateManageProfile.model.$params).find(key => vuelidateManageProfile.model[key].$invalid)
                    //@ts-ignore
                    if (invalidField && this.$refs.manageProfileForm.$refs[invalidField]) {
                        //@ts-ignore
                        this.$refs.manageProfileForm.$refs[invalidField].$refs.input.focus()
                    }
                }
            }
            else {
                // No changes detected
                this.showSnackBar('Save Successful', true)
            }

            this.nextButtonKey++
        }

        async showVerificationCodeForm() {
            // @ts-ignore
            await this.sendTokenAsync(
                axiosInstanceCommon,
                this.manageProfileModel.entityId,
                false,
                true,
                this.manageProfileModel.mobilePhoneNumber,
                false);
        }

        public async saveChanges(): Promise<void> {
            axiosInstance.post('Account/save-profile', this.manageProfileModel).then(
                async () => {
                    var isMfaEnabled = this.$store.state.FeatureFlags.featureFlagsData.MultiFactorAuthentication;
                    if (this.mobilePhoneNumberInitialValue != this.manageProfileModel.mobilePhoneNumber && isMfaEnabled == true) {
                        sessionStorage.mfaForm = MfaForm.VerificationCodeForm;
                        sessionStorage.isManageProfile = true
                        await this.showVerificationCodeForm()
                        // @ts-ignore
                        this.navigator.goToHome();
                    }
                    await this.loadProfile(() => {
                        this.manageProfileModel.tcpaWarningText = ''
                        this.showSnackBar('Save Successful', true)
                    })
                })
        }

        public async created(): Promise<void> {
            this.setTitle(routeConstants.manageAccount.display)
            this.$store.dispatch('Account/setPassordReuseError', false)
            // ADA Performance
            this.isChromeBrowser = /Chrome/.test(navigator.userAgent) && /Google Inc/.test(navigator.vendor)

            await this.loadDashboardBanner(async () => {
                await this.loadProfile(async () => {
                    await this.loadChangePassword(() => {
                        this.loaded = true
                    })
                })
            })
        }

        public async loadProfile(callback: Function): Promise<void> {
            await this.$store.dispatch('Account/getManageProfile').then(() => {
                this.isTcpaInitialValue = this.$store.state.Account.profile.isTcpa
                this.mobilePhoneNumberInitialValue = this.$store.state.Account.profile.mobilePhoneNumber
                this.tcpaWarningText = this.$store.state.Account.profile.tcpaWarningText
                this.manageProfileModel = this.$store.state.Account.profile

                if (callback != null) {
                    callback()
                }
            })
        }

        public async loadChangePassword(callback: Function): Promise<void> {
            await this.$store.dispatch('Account/getChangePassword').then(() => {
                var pwValidation = this.$store.state.Account.changePassword.settingPassword
                this.changePasswordForm.usePwComplexity = pwValidation.usePwComplexity == 1
                this.changePasswordForm.pwDisallowReuse = pwValidation.pwDisallowReuse == 1
                this.changePasswordForm.passwordValidationSetting = {
                    pwLowercaseLetter: pwValidation.pwLowercaseLetter,
                    pwMinChars: pwValidation.pwMinChars,
                    pwNumber: pwValidation.pwNumber,
                    pwSpecialCharacters: pwValidation.pwSpecialCharacters,
                    pwUppercaseLetter: pwValidation.pwSpecialCharacters
                }

                if (callback != null) {
                    callback()
                }
            })
        }

        public async loadDashboardBanner(callback: Function): Promise<void> {
            await this.$store.dispatch('Dashboard/getDashboardBanner').then(() => {
                this.dashboardBanner = this.$store.state.Dashboard.dashboardBanner
                this.dashboardBanner.subHeader = ""

                if (callback != null) {
                    callback()
                }
            })
        }

        public changePasswordButtonAction(): void {
            var dataPost = {
                entityId: this.$store.state.Account.changePassword.entityId,
                currentPassword: this.changePasswordForm.currentPassword,
                newPassword: this.changePasswordForm.newPassword,
                confirmPassword: this.changePasswordForm.confirmPassword,
            }
            this.$store.dispatch('Account/setPassordReuseError', false)
            axiosInstance.post('account/change-password', dataPost)
                .then(
                    response => {
                        this.showAlertDialog = true
                        this.verifyCurrentPasswordADA = response.data.message
                        this.isChangedSuccess = true
                        this.$nextTick(() => {
                            setTimeout(() => {
                                document.getElementById("btnOK")?.focus()
                            }, 100)
                        })
                    },
                    error => {
                        switch (error.response.status) {
                            case 400:
                                if (error.response.data.passwordReuseError) {
                                    this.$store.dispatch('Account/setPassordReuseError', true)
                                    this.iterationErrorMessageADA = ''
                                    setTimeout(() => {
                                        this.iterationErrorMessageADA = this.iterationErrorMessage
                                    }, 500)

                                    if (document.getElementById("new-password")) {
                                        document.getElementById("new-password")?.focus()
                                    }

                                    break;
                                }
                                this.showAlertDialog = true
                                this.verifyCurrentPasswordADA = 'Alert dialog. '
                                if (error.response.data.errorType === 'LoginLocked') {
                                    this.verifyCurrentPasswordADA = error.response.data.message
                                    this.isChangedSuccess = false
                                    this.isAccountLocked = true
                                }

                                if (error.response.data.message) {
                                    this.verifyCurrentPasswordADA = error.response.data.message
                                    this.isChangedSuccess = false
                                } else {
                                    this.verifyCurrentPasswordADA = 'ERROR!'
                                    this.isChangedSuccess = false
                                }

                                break
                        }
                    },
                )
        }

        public processOKDialog(): void {
            this.showAlertDialog = false
            this.verifyCurrentPasswordADA = ''
            this.titleAlert = ''
            if (this.isChangedSuccess) {
                sessionStorage.isPasswordExpired = false
                //@ts-ignore
                this.navigator.goToDashboard();
                return;
            }
            if (this.isAccountLocked) {
                //@ts-ignore
                this.navigator.goToLogout();
            }
        }

        @Watch('manageProfileModel.isTcpa', { immediate: false, deep: true })
        isTcpaChanged(newValue: boolean): void {
            if (newValue && newValue != this.isTcpaInitialValue) {
                this.manageProfileModel.tcpaWarningText = this.tcpaWarningText

            }
            else if (this.manageProfileModel.mobilePhoneNumber == this.mobilePhoneNumberInitialValue
                && this.manageProfileModel.isTcpa == this.isTcpaInitialValue) {
                this.manageProfileModel.tcpaWarningText = ''
            }
        }

        @Watch('manageProfileModel.mobilePhoneNumber', { immediate: false, deep: true })
        isMobilePhoneChanged(newValue: string): void {
            if (newValue != this.mobilePhoneNumberInitialValue) {
                this.manageProfileModel.tcpaWarningText = this.tcpaWarningText
            }
            else if (this.manageProfileModel.mobilePhoneNumber == this.mobilePhoneNumberInitialValue
                && this.manageProfileModel.isTcpa == this.isTcpaInitialValue) {
                this.manageProfileModel.tcpaWarningText = ''
            }
        }
    }
</script>

<style scoped>
    .container >>> .tcpaOptionForm {
        padding: 0px 18px;
        border-radius: 3px;
        border: solid 2px rgba(var(--color-secondary-btn-outline));
        background-color: rgba(var(--color-page-background));
        border-style: solid;
        border-color: rgba(var(--color-secondary-btn-outline));
        margin-bottom: 63px;
    }

    .tcpaOptionForm >>> .tcpaCheckbox-label {
        font-size: 14px !important;
        font-weight: normal !important;
        font-stretch: normal !important;
        font-style: normal !important;
        line-height: 2.14 !important;
        letter-spacing: normal !important;
    }

    .return-to-dashboard {
        font-size: 16px;
        font-weight: 500;
        font-stretch: normal;
        font-style: normal;
        line-height: normal;
        letter-spacing: 1.2px;
        font-family: 'Whitney Semi';
        color: rgba(var(--color-icon-tag-text));
        margin-left: 10px;
    }

    .link {
        text-decoration: none;
        color: rgba(var(--color-icon-tag-text)) !important;
    }

    .containter-table {
        width: 100%;
    }

    .reuse-error {
        font-size: x-large;
        font-weight: bold;
        max-width: 763px;
    }

    .div-content {
        display: flex;
        flex-wrap: wrap;
        position: relative;
    }

    .bottomLine {
        height: 1px;
        width: 100%;
        border-bottom: solid rgba(var(--color-structural-ui-light)) 1px;
        position: absolute;
        margin-top: 48px;
    }
</style>