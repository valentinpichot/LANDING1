<script setup lang="ts">
import { ref, computed } from 'vue';
import { useI18n } from 'vue-i18n';

const { t } = useI18n();

// Set VITE_WEB3FORMS_KEY in .env — get a free key at web3forms.com
// In demo mode (no key), the form simulates a successful submission.
const WEB3FORMS_KEY = (import.meta as any).env?.VITE_WEB3FORMS_KEY ?? '';

const TOTAL_STEPS = 3;
const step = ref(1);
const stepDirection = ref<'forward' | 'back'>('forward');

const form = ref({ name: '', email: '', projectType: '', budget: '', message: '' });
const fieldErrors = ref({ name: false, email: false });
const stepTwoError = ref(false);
const isSubmitting = ref(false);
const submitState = ref<'idle' | 'success' | 'error'>('idle');

const charCount = computed(() => form.value.message.length);

const projectTypes = [
  { key: 'website',   icon: '↗' },
  { key: 'ecommerce', icon: '◈' },
  { key: 'webapp',    icon: '{ }' },
  { key: 'redesign',  icon: '↺' },
];
const budgetKeys = ['under3k', 'mid3to8k', 'mid8to15k', 'above15k'];

function validateStep1(): boolean {
  const emailRe = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  fieldErrors.value.name  = form.value.name.trim().length < 2;
  fieldErrors.value.email = !emailRe.test(form.value.email.trim());
  return !fieldErrors.value.name && !fieldErrors.value.email;
}

function next() {
  if (step.value === 1 && !validateStep1()) return;
  if (step.value === 2) {
    stepTwoError.value = !form.value.projectType || !form.value.budget;
    if (stepTwoError.value) return;
  }
  stepDirection.value = 'forward';
  step.value++;
}

function back() {
  stepDirection.value = 'back';
  step.value--;
}

async function submit() {
  isSubmitting.value = true;
  submitState.value = 'idle';
  try {
    if (!WEB3FORMS_KEY) {
      await new Promise(r => setTimeout(r, 1400));
      submitState.value = 'success';
      return;
    }
    const payload = {
      access_key: WEB3FORMS_KEY,
      name: form.value.name,
      email: form.value.email,
      subject: `[Vision Pure] Nouveau projet — ${t(`contact.project_${form.value.projectType}`)}`,
      message: [
        `Nom : ${form.value.name}`,
        `Email : ${form.value.email}`,
        `Type : ${t(`contact.project_${form.value.projectType}`)}`,
        `Budget : ${t(`contact.budget_${form.value.budget}`)}`,
        `Message :\n${form.value.message || '—'}`,
      ].join('\n'),
    };
    const res = await fetch('https://api.web3forms.com/submit', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', Accept: 'application/json' },
      body: JSON.stringify(payload),
    });
    const data = await res.json();
    submitState.value = data.success ? 'success' : 'error';
  } catch {
    submitState.value = 'error';
  } finally {
    isSubmitting.value = false;
  }
}
</script>

<template>
  <div class="contact__layout">
    <!-- ── Info column ───────────────────────────────────────────── -->
    <div class="contact__info">
      <div class="info-block">
        <span class="info-label reveal-text">{{ t('contact.email_label') }}</span>
        <a href="mailto:hello@visionpure.studio" class="info-value reveal-text">hello@visionpure.studio</a>
      </div>
      <div class="info-block">
        <span class="info-label reveal-text">{{ t('contact.location_label') }}</span>
        <p class="info-value reveal-text">{{ t('contact.location_line1') }}<br />{{ t('contact.location_line2') }}</p>
      </div>
      <div class="info-block">
        <span class="info-label reveal-text">{{ t('contact.availability_label') }}</span>
        <p class="info-value reveal-text">{{ t('contact.availability_line1') }}<br />{{ t('contact.availability_line2') }}</p>
      </div>
      <div class="info-block">
        <span class="info-label reveal-text">{{ t('contact.social_label') }}</span>
        <div class="social-links reveal-text">
          <a href="#">Instagram</a>
          <a href="#">LinkedIn</a>
          <a href="#">Dribbble</a>
        </div>
      </div>
    </div>

    <!-- ── Multi-step form ───────────────────────────────────────── -->
    <div class="contact__form-card reveal-text">

      <!-- Success -->
      <Transition name="fade" mode="out-in">
        <div v-if="submitState === 'success'" class="form-success">
          <div class="form-success__ring">
            <span class="form-success__check">✓</span>
          </div>
          <h3>{{ t('contact.success_title') }}</h3>
          <p>{{ t('contact.success_text') }}</p>
        </div>

        <div v-else class="form-inner">
          <!-- Step progress -->
          <div class="form-stepper">
            <template v-for="s in TOTAL_STEPS" :key="s">
              <div class="stepper__node">
                <div
                  class="stepper__circle"
                  :class="{ 'is-done': s < step, 'is-active': s === step }"
                >
                  <span v-if="s < step">✓</span>
                  <span v-else>{{ s }}</span>
                </div>
                <span class="stepper__label" :class="{ 'is-active': s === step }">
                  {{ t(`contact.step${s}_title`) }}
                </span>
              </div>
              <div v-if="s < TOTAL_STEPS" class="stepper__line">
                <div class="stepper__line-fill" :class="{ filled: s < step }"></div>
              </div>
            </template>
          </div>

          <!-- Step content with slide transition -->
          <Transition
            :name="stepDirection === 'forward' ? 'slide-fwd' : 'slide-back'"
            mode="out-in"
          >
            <!-- Step 1 — Identity -->
            <div v-if="step === 1" key="s1" class="form-step">
              <div class="form-group" :class="{ 'has-error': fieldErrors.name }">
                <input
                  type="text" id="cf-name" v-model="form.name" placeholder=" "
                  autocomplete="name"
                  @input="fieldErrors.name = false"
                />
                <label for="cf-name">{{ t('contact.form_name') }}</label>
                <span v-if="fieldErrors.name" class="field-error">{{ t('contact.error_required') }}</span>
              </div>
              <div class="form-group" :class="{ 'has-error': fieldErrors.email }">
                <input
                  type="email" id="cf-email" v-model="form.email" placeholder=" "
                  autocomplete="email"
                  @input="fieldErrors.email = false"
                />
                <label for="cf-email">{{ t('contact.form_email') }}</label>
                <span v-if="fieldErrors.email" class="field-error">{{ t('contact.error_email') }}</span>
              </div>
              <div class="form-actions">
                <button class="btn-next" type="button" @click="next">
                  {{ t('contact.form_next') }} <span class="btn-arrow">→</span>
                </button>
              </div>
            </div>

            <!-- Step 2 — Project type + budget -->
            <div v-else-if="step === 2" key="s2" class="form-step">
              <p class="step-label">{{ t('contact.project_type_label') }}</p>
              <div class="project-grid">
                <button
                  v-for="pt in projectTypes" :key="pt.key" type="button"
                  class="project-card"
                  :class="{ 'project-card--on': form.projectType === pt.key }"
                  @click="form.projectType = pt.key; stepTwoError = false"
                >
                  <span class="project-card__icon">{{ pt.icon }}</span>
                  <span class="project-card__name">{{ t(`contact.project_${pt.key}`) }}</span>
                </button>
              </div>

              <p class="step-label">{{ t('contact.budget_label') }}</p>
              <div class="budget-grid">
                <button
                  v-for="bk in budgetKeys" :key="bk" type="button"
                  class="budget-pill"
                  :class="{ 'budget-pill--on': form.budget === bk }"
                  @click="form.budget = bk; stepTwoError = false"
                >
                  {{ t(`contact.budget_${bk}`) }}
                </button>
              </div>

              <p v-if="stepTwoError" class="step-error">{{ t('contact.error_step2') }}</p>

              <div class="form-actions form-actions--split">
                <button class="btn-back" type="button" @click="back">
                  <span class="btn-arrow">←</span> {{ t('contact.form_back') }}
                </button>
                <button class="btn-next" type="button" @click="next">
                  {{ t('contact.form_next') }} <span class="btn-arrow">→</span>
                </button>
              </div>
            </div>

            <!-- Step 3 — Message + submit -->
            <div v-else key="s3" class="form-step">
              <div class="form-group textarea-group">
                <textarea
                  id="cf-msg" v-model="form.message" placeholder=" "
                  maxlength="500" rows="5"
                ></textarea>
                <label for="cf-msg">{{ t('contact.form_message') }}</label>
                <span
                  class="char-count"
                  :class="{ 'near-limit': charCount > 400, 'at-limit': charCount >= 500 }"
                >{{ charCount }}/500</span>
              </div>

              <p v-if="submitState === 'error'" class="step-error">{{ t('contact.error_submit') }}</p>

              <div class="form-actions form-actions--split">
                <button class="btn-back" type="button" @click="back">
                  <span class="btn-arrow">←</span> {{ t('contact.form_back') }}
                </button>
                <button
                  class="submit-btn" type="button"
                  :disabled="isSubmitting"
                  @click="submit"
                >
                  <span v-if="!isSubmitting">{{ t('contact.form_submit') }}</span>
                  <span v-else class="btn-spinner"></span>
                  <div class="btn-glow"></div>
                </button>
              </div>
            </div>
          </Transition>
        </div>
      </Transition>
    </div>
  </div>
</template>

<style lang="scss">
/* ── Layout ─────────────────────────────────────────────────────── */
.contact__layout {
  display: grid;
  grid-template-columns: 1fr 1.5fr;
  gap: clamp(4rem, 10vw, 8rem);

  @media (max-width: 992px) {
    grid-template-columns: 1fr;
    gap: 4rem;
  }
}

/* ── Info column ─────────────────────────────────────────────────── */
.contact__info {
  display: flex;
  flex-direction: column;
  gap: 3rem;

  .info-block {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
  }

  .info-label {
    font-size: 0.8rem;
    text-transform: uppercase;
    letter-spacing: 0.2em;
    color: rgba(255, 255, 255, 0.4);
  }

  .info-value {
    font-size: clamp(1.1rem, 1.8vw, 1.6rem);
    font-weight: 500;
    color: #fff;
    text-decoration: none;
    transition: color 0.3s ease;
    line-height: 1.4;

    &:hover { color: var(--color-accent); }
  }

  .social-links {
    display: flex;
    gap: 1.75rem;
    flex-wrap: wrap;

    a {
      color: rgba(255, 255, 255, 0.5);
      text-decoration: none;
      font-size: 1rem;
      transition: color 0.3s ease;
      &:hover { color: var(--color-accent); }
    }
  }
}

/* ── Form card ───────────────────────────────────────────────────── */
.contact__form-card {
  background: rgba(255, 255, 255, 0.02);
  border: 1px solid rgba(255, 255, 255, 0.07);
  border-radius: 24px;
  padding: clamp(2rem, 5vw, 2.75rem);
  position: relative;
  overflow: hidden;

  &::before {
    content: '';
    position: absolute;
    top: 0; left: 10%; right: 10%;
    height: 1px;
    background: linear-gradient(90deg, transparent, rgba(0, 255, 204, 0.35), transparent);
  }
}

/* ── Stepper ─────────────────────────────────────────────────────── */
.form-stepper {
  display: flex;
  align-items: flex-start;
  margin-bottom: 2.25rem;
}

.stepper__node {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.45rem;
  flex-shrink: 0;
}

.stepper__circle {
  width: 30px;
  height: 30px;
  border-radius: 50%;
  border: 1px solid rgba(255, 255, 255, 0.15);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.65rem;
  font-weight: 700;
  color: rgba(255, 255, 255, 0.3);
  transition: all 0.35s cubic-bezier(0.25, 1, 0.5, 1);

  &.is-active {
    border-color: var(--color-accent);
    color: var(--color-accent);
    box-shadow: 0 0 14px rgba(0, 255, 204, 0.25);
  }

  &.is-done {
    background: var(--color-accent);
    border-color: var(--color-accent);
    color: #000;
  }
}

.stepper__label {
  font-size: 0.58rem;
  text-transform: uppercase;
  letter-spacing: 0.12em;
  color: rgba(255, 255, 255, 0.25);
  transition: color 0.3s ease;
  white-space: nowrap;

  &.is-active { color: var(--color-accent); }
}

.stepper__line {
  flex: 1;
  height: 1px;
  background: rgba(255, 255, 255, 0.08);
  margin: 15px 0.5rem 0;
  position: relative;
  overflow: hidden;
}

.stepper__line-fill {
  position: absolute;
  top: 0; left: 0;
  height: 100%;
  width: 0;
  background: var(--color-accent);
  transition: width 0.55s cubic-bezier(0.25, 1, 0.5, 1);

  &.filled { width: 100%; }
}

/* ── Steps ───────────────────────────────────────────────────────── */
.form-step {
  min-height: 280px;
  display: flex;
  flex-direction: column;
  gap: 1.4rem;
}

/* ── Inputs ──────────────────────────────────────────────────────── */
.form-group {
  position: relative;

  input, textarea {
    width: 100%;
    padding: 1rem 0;
    background: transparent;
    border: none;
    border-bottom: 1px solid rgba(255, 255, 255, 0.1);
    color: #fff;
    font-family: inherit;
    font-size: 1rem;
    outline: none;
    transition: border-color 0.3s ease;
    resize: none;

    &:focus { border-color: var(--color-accent); }

    &:focus + label,
    &:not(:placeholder-shown) + label {
      transform: translateY(-22px);
      font-size: 0.72rem;
      color: var(--color-accent);
    }
  }

  label {
    position: absolute;
    left: 0;
    top: 1rem;
    color: rgba(255, 255, 255, 0.35);
    font-size: 0.95rem;
    pointer-events: none;
    transition: all 0.28s cubic-bezier(0.25, 1, 0.5, 1);
  }

  &.has-error input,
  &.has-error textarea {
    border-color: rgba(255, 80, 80, 0.6);
  }
}

.field-error {
  display: block;
  margin-top: 0.35rem;
  font-size: 0.7rem;
  color: rgba(255, 100, 100, 0.9);
  letter-spacing: 0.05em;
}

.textarea-group {
  position: relative;

  textarea { padding-bottom: 1.75rem; }
}

.char-count {
  position: absolute;
  bottom: 0.5rem;
  right: 0;
  font-size: 0.64rem;
  color: rgba(255, 255, 255, 0.2);
  transition: color 0.2s ease;
  font-variant-numeric: tabular-nums;

  &.near-limit { color: rgba(255, 160, 60, 0.8); }
  &.at-limit   { color: rgba(255, 70, 70, 0.9); }
}

/* ── Step 2 selectors ────────────────────────────────────────────── */
.step-label {
  font-size: 0.68rem;
  text-transform: uppercase;
  letter-spacing: 0.2em;
  color: rgba(255, 255, 255, 0.35);
  margin-bottom: -0.5rem;
}

.project-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 0.65rem;
}

.project-card {
  background: rgba(255, 255, 255, 0.02);
  border: 1px solid rgba(255, 255, 255, 0.07);
  border-radius: 14px;
  padding: 1.1rem 1rem;
  display: flex;
  flex-direction: column;
  gap: 0.55rem;
  cursor: pointer;
  text-align: left;
  font-family: inherit;
  transition: all 0.22s ease;

  &__icon {
    font-size: 1rem;
    color: rgba(255, 255, 255, 0.25);
    line-height: 1;
    transition: color 0.22s ease;
    font-style: normal;
  }

  &__name {
    font-size: 0.78rem;
    font-weight: 600;
    color: rgba(255, 255, 255, 0.45);
    letter-spacing: 0.02em;
    transition: color 0.22s ease;
  }

  &:hover {
    border-color: rgba(0, 255, 204, 0.2);
    background: rgba(0, 255, 204, 0.03);
    .project-card__icon, .project-card__name { color: rgba(255, 255, 255, 0.8); }
  }

  &--on {
    border-color: var(--color-accent) !important;
    background: rgba(0, 255, 204, 0.06) !important;
    box-shadow: 0 0 18px rgba(0, 255, 204, 0.07);
    .project-card__icon { color: var(--color-accent) !important; }
    .project-card__name { color: #fff !important; }
  }
}

.budget-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 0.6rem;
}

.budget-pill {
  padding: 0.65rem 0.75rem;
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 10px;
  background: transparent;
  color: rgba(255, 255, 255, 0.38);
  font-family: inherit;
  font-size: 0.76rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s ease;
  text-align: center;

  &:hover {
    border-color: rgba(0, 255, 204, 0.2);
    color: rgba(255, 255, 255, 0.7);
  }

  &--on {
    border-color: var(--color-accent) !important;
    color: var(--color-accent) !important;
    background: rgba(0, 255, 204, 0.05) !important;
    box-shadow: 0 0 10px rgba(0, 255, 204, 0.05);
  }
}

.step-error {
  font-size: 0.72rem;
  color: rgba(255, 100, 100, 0.85);
  letter-spacing: 0.04em;
  margin-top: -0.5rem;
}

/* ── Buttons ─────────────────────────────────────────────────────── */
.form-actions {
  display: flex;
  justify-content: flex-end;
  margin-top: auto;
  padding-top: 0.5rem;

  &--split {
    justify-content: space-between;
    align-items: center;
  }
}

.btn-next {
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.75rem 2rem;
  background: #fff;
  color: #000;
  border: none;
  border-radius: 100px;
  font-family: inherit;
  font-size: 0.8rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.25, 1, 0.5, 1);

  .btn-arrow { transition: transform 0.3s ease; }

  &:hover {
    transform: translateY(-2px);
    box-shadow: 0 8px 24px rgba(255, 255, 255, 0.12);
    .btn-arrow { transform: translateX(3px); }
  }
}

.btn-back {
  display: inline-flex;
  align-items: center;
  gap: 0.4rem;
  padding: 0.5rem 0;
  background: transparent;
  border: none;
  color: rgba(255, 255, 255, 0.35);
  font-family: inherit;
  font-size: 0.75rem;
  font-weight: 500;
  text-transform: uppercase;
  letter-spacing: 0.1em;
  cursor: pointer;
  transition: color 0.25s ease;

  .btn-arrow { transition: transform 0.3s ease; }

  &:hover {
    color: rgba(255, 255, 255, 0.7);
    .btn-arrow { transform: translateX(-3px); }
  }
}

.submit-btn {
  align-self: flex-end;
  padding: 0.85rem 2.5rem;
  background: #fff;
  color: #000;
  border: none;
  border-radius: 100px;
  font-family: inherit;
  font-size: 0.8rem;
  font-weight: 700;
  cursor: pointer;
  position: relative;
  overflow: hidden;
  transition: transform 0.3s ease;

  &:disabled { opacity: 0.7; cursor: not-allowed; transform: none; }

  &:not(:disabled):hover {
    transform: translateY(-3px);
    .btn-glow { opacity: 1; }
  }

  span { position: relative; z-index: 2; }

  .btn-glow {
    position: absolute;
    inset: 0;
    background: var(--color-accent-gradient);
    opacity: 0;
    transition: opacity 0.3s ease;
    z-index: 1;
  }
}

.btn-spinner {
  display: inline-block;
  width: 16px;
  height: 16px;
  border: 2px solid rgba(0, 0, 0, 0.2);
  border-top-color: #000;
  border-radius: 50%;
  animation: cf-spin 0.65s linear infinite;
  position: relative;
  z-index: 2;
}

@keyframes cf-spin {
  to { transform: rotate(360deg); }
}

/* ── Success state ───────────────────────────────────────────────── */
.form-success {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  text-align: center;
  gap: 1.25rem;
  min-height: 340px;

  &__ring {
    width: 64px;
    height: 64px;
    border-radius: 50%;
    border: 2px solid var(--color-accent);
    display: flex;
    align-items: center;
    justify-content: center;
    box-shadow: 0 0 30px rgba(0, 255, 204, 0.2);
    animation: success-pop 0.55s cubic-bezier(0.34, 1.56, 0.64, 1) both;
  }

  &__check {
    font-size: 1.5rem;
    color: var(--color-accent);
    animation: success-pop 0.55s 0.1s cubic-bezier(0.34, 1.56, 0.64, 1) both;
  }

  h3 {
    font-size: 1.4rem;
    font-weight: 700;
    color: #fff;
    letter-spacing: -0.02em;
  }

  p {
    font-size: 0.95rem;
    color: rgba(255, 255, 255, 0.45);
    line-height: 1.6;
  }
}

@keyframes success-pop {
  from { transform: scale(0.4); opacity: 0; }
  to   { transform: scale(1);   opacity: 1; }
}

/* ── Transitions ─────────────────────────────────────────────────── */
.slide-fwd-enter-active,
.slide-fwd-leave-active,
.slide-back-enter-active,
.slide-back-leave-active {
  transition: all 0.3s cubic-bezier(0.25, 1, 0.5, 1);
}

.slide-fwd-enter-from  { opacity: 0; transform: translateX(22px); }
.slide-fwd-leave-to    { opacity: 0; transform: translateX(-22px); }
.slide-back-enter-from { opacity: 0; transform: translateX(-22px); }
.slide-back-leave-to   { opacity: 0; transform: translateX(22px); }

.fade-enter-active, .fade-leave-active { transition: opacity 0.4s ease; }
.fade-enter-from, .fade-leave-to       { opacity: 0; }
</style>
