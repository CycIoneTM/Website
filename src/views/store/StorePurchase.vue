<template>
	<main v-if="actor.id" class="store-purchase" :lock="waiting">
		<span v-if="waiting" class="lock-notice"> {{ t("store.payment_popup_cta") }} </span>

		<h2>
			<router-link :to="{ name: 'Store' }">
				<Icon icon="arrow-left" />
			</router-link>
			{{ t("store.billing_information_heading") }}
		</h2>
		<p></p>
		<BillingForm @update-form="formData = $event" />

		<h2 v-if="gift">{{ t("store.payment_gift_heading") }}</h2>
		<div v-if="gift" class="gift-recipient">
			<UserQuickSearch v-if="!recipient" :event-only="true" @select="recipient = $event" />
			<div v-else class="selected-recipient">
				<UserTag :user="recipient" scale="1.5rem" />
				<span @click="recipient = null">
					<Icon icon="close" />
				</span>
			</div>
		</div>

		<h2>{{ t("store.payment_information_heading") }}</h2>

		<!-- Show Payment Methods -->
		<p>{{ t("store.payment_methods_hint") }}</p>
		<div class="payment-methods">
			<span
				v-for="pm of paymentMethods"
				:key="pm.id"
				:class="{ selected: pm.id === selectedMethod }"
				@click="selectedMethod = pm.id"
			>
				<Icon
					v-tooltip="`Pay via ${pm.name}`"
					v-tooltip:position="'top'"
					:lib="pm.lib"
					size="xl"
					:icon="pm.icon"
				/>
			</span>
		</div>

		<!-- Product settings -->
		<div v-if="egv.currentProduct?.name === 'subscription'" class="product-display" product="subscription">
			<p>
				<span :style="{ color: 'orange' }">
					<Logo />
				</span>
				{{ t("store.product_type_subscription").toUpperCase() }}
			</p>
			<span v-if="egv.currentPlan">
				<span>{{ egv.currentPlan.interval }} {{ egv.currentPlan.interval_unit }}</span>
				<span> €{{ Number(egv.currentPlan.price) / 100 }} </span>
			</span>
		</div>

		<Button
			:disabled="!selectedMethod || !formData || (gift && !recipient)"
			color="primary"
			:label="t('store.pay_button', [t(`store.payment_method_${selectedMethod}`)])"
			@click="checkout"
		/>
	</main>

	<main v-else class="store-purchase no-auth">
		<h2>Please sign in to purchase</h2>
		<div>
			<LoginButton />
		</div>
	</main>
</template>

<script setup lang="ts">
import { defineAsyncComponent, onUnmounted, ref } from "vue";
import { useI18n } from "vue-i18n";
import { useRoute, useRouter } from "vue-router";
import { useActor } from "@/store/actor";
import { LocalStorageKeys } from "@/store/lskeys";
import { useModal } from "@/store/modal";
import { User } from "@/structures/User";
import PurchaseSuccessModal from "@/views/store/PurchaseSuccessModal.vue";
import Logo from "@/components/base/Logo.vue";
import Button from "@/components/utility/Button.vue";
import Icon from "@/components/utility/Icon.vue";
import LoginButton from "@/components/utility/LoginButton.vue";
import UserTag from "@/components/utility/UserTag.vue";
import BillingForm from "./BillingForm.vue";
import { EgVault, useEgVault } from "./egvault";

const UserQuickSearch = defineAsyncComponent(() => import("@/components/utility/UserQuickSearch.vue"));

const { t } = useI18n();
const actor = useActor();
const route = useRoute();
const egv = useEgVault();

const gift: boolean = route.query.gift === "1";

const selectedMethod = ref("stripe");
const paymentMethods = ref([
	{ id: "stripe", name: "Credit Card", icon: "credit-card" },
	{ id: "paypal", name: "PayPal", icon: "cc-paypal", lib: "fab" },
] as PaymentMethod[]);

const formData = ref("");

const recipient = ref<User | null>(null);

const waiting = ref(false);
const winID = Math.random().toString(36).substring(2);
const checkout = async () => {
	if (!egv.currentPlan) return;

	waiting.value = true;

	const win = window.open(
		"about:blank",
		"SEVENTV_CHECKOUT:" + winID,
		"_blank, width=1200, height=900, menubar=no, location=no",
	);

	let renewInterval = "";
	switch (egv.currentPlan.interval_unit) {
		case "MONTH":
			renewInterval = "monthly";
			break;
		case "YEAR":
			renewInterval = "yearly";
			break;
	}

	const fd = JSON.parse(formData.value);

	const giftQuery = gift && recipient.value ? `&gift_for=${recipient.value.id}` : "";
	const resp = await fetch(
		`${EgVault.api}/v1/subscriptions?renew_interval=${renewInterval}&payment_method=${selectedMethod.value}&next=true${giftQuery}`,
		{
			method: "POST",
			body: JSON.stringify({
				prefill: {
					first_name: fd.firstName,
					last_name: fd.lastName,
					email: fd.email,
				},
			}),
			headers: {
				Authorization: `Bearer ${localStorage.getItem(LocalStorageKeys.TOKEN)}`,
			},
		},
	);
	if (!resp || !resp.ok) {
		waiting.value = false;

		egv.showError(resp);
		win?.close();

		return undefined;
	}

	const data = await resp.json();
	win!.location.href = data.url;

	const i = setInterval(() => {
		if (!win || win?.closed) {
			clearInterval(i);
			waiting.value = false;
		}
	}, 1000);
};

const onExit = () => true;
window.onbeforeunload = onExit;

const router = useRouter();
const modal = useModal();

// Waiting for message from transaction window
const onMessage = (ev: MessageEvent) => {
	const w = ev.target as Window;
	if (!w) {
		return undefined;
	}

	const data = JSON.parse(ev.data);
	if (!data || data.id !== winID) {
		return;
	}

	if (data.status === "COMPLETE") {
		modal.open("purchase-success", {
			component: PurchaseSuccessModal,
			events: {},
			props: {},
		});
	}

	router.replace({ name: "Store", force: true });
};
window.addEventListener("message", onMessage);

onUnmounted(() => {
	window.onbeforeunload = null;
	window.removeEventListener("message", onMessage);
});

interface PaymentMethod {
	id: string;
	name: string;
	icon: string;
	lib?: string;
	selected?: boolean;
}
</script>

<style scoped lang="scss">
@import "@scss/themes.scss";

main.store-purchase {
	margin-top: 1em;
	margin-left: 15%;
	margin-right: 15%;
	padding: 1em;
	border-radius: 0.4em;

	display: flex;
	flex-direction: column;

	@include themify {
		background-color: darken(themed("backgroundColor"), 2);

		> div.product-display {
			background-color: themed("backgroundColor");
		}
	}

	&[lock="true"] {
		> span.lock-notice {
			z-index: 1;
			width: 100%;
			text-align: center;
			font-size: 1.5em;
		}
		> *:not(span.lock-notice) {
			opacity: 0.25;
		}
		user-select: none;
		pointer-events: none;
	}

	> h2 {
		width: 100%;
		text-align: start;
	}

	> div.gift-recipient {
		margin: 1em;

		> .selected-recipient {
			display: flex;
			align-items: center;
			gap: 0.25em;
		}
	}

	> div.payment-methods {
		display: flex;
		margin: 1em;
		gap: 0.5em;

		> span {
			cursor: pointer;
			font-size: 3em;

			&.selected {
				color: rgb(160, 255, 160);
			}
		}
	}

	> div.product-display {
		width: fit-content;
		padding: 0.66em;
		margin: 0.5em;
		margin-bottom: 1em;

		> p {
			font-size: 1.15em;
			font-weight: 600;
		}
		> span {
			display: flex;
			justify-content: space-between;
			text-transform: capitalize;
		}
	}

	&.no-auth {
		font-size: 1.5em;
		margin-left: 20%;
		margin-right: 20%;

		gap: 1em;
	}

	@media screen and (max-width: 600px) {
		margin: 0 !important;
	}
}
</style>
