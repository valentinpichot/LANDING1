<script setup lang="ts">
import { ref, computed, watch, reactive, onMounted, onUnmounted } from 'vue';
import { useI18n } from 'vue-i18n';
import ContactForm from './components/ContactForm.vue';
import gsap from 'gsap';
import ScrollTrigger from 'gsap/ScrollTrigger';
import Lenis from '@studio-freight/lenis';

gsap.registerPlugin(ScrollTrigger);

const { t, tm, locale } = useI18n();

// Persist locale, sync html[lang] and document.title (#8, #13)
watch(locale, (val) => {
  localStorage.setItem('locale', val as string);
  document.documentElement.lang = val as string;
  document.title = t('meta.title');
}, { immediate: true });

// ── Refs (dead refs removed: gallerySection, contactSection) (#10) ──────────
const mainContainer      = ref<HTMLElement | null>(null);
const heroSection        = ref<HTMLElement | null>(null);
const heroTitle          = ref<HTMLElement | null>(null);
const heroBtns           = ref<HTMLElement | null>(null);
const processSection     = ref<HTMLElement | null>(null);
const processCardsWrapper = ref<HTMLElement | null>(null);
const galleryImages      = ref<HTMLElement[]>([]);
const navbar             = ref<HTMLElement | null>(null);
const cursor             = ref<HTMLElement | null>(null);

// ── Mobile menu (#3, #4) ─────────────────────────────────────────────────────
const menuOpen = ref(false);

const toggleMenu = () => {
  menuOpen.value = !menuOpen.value;
  // Freeze / resume Lenis behind the drawer (#3)
  if (menuOpen.value) lenis?.stop();
  else lenis?.start();
};

const closeMenu = () => {
  menuOpen.value = false;
  lenis?.start(); // always restart Lenis on close (#3)
};

// ── Preloader ────────────────────────────────────────────────────────────────
const preloaderDone = ref(false);

// ── Hero stat counters ───────────────────────────────────────────────────────
const heroStats = reactive({ clients: 0, projects: 0 });

// ── Hero title as computed chars — no longer hardcoded HTML (#15) ────────────
const STUDIO_NAME   = 'V-STUDIOS';
const STUDIO_SUFFIX = 'DEMO';
const heroWords = computed(() =>
  [STUDIO_NAME, STUDIO_SUFFIX].map(word => [...word])
);

// ── Gallery data ─────────────────────────────────────────────────────────────
const galleryData = [
  { img: 'https://images.unsplash.com/photo-1618221195710-dd6b41faaea6?q=80&w=800&h=1200&auto=format&fit=crop', year: '2025', width: 800,  height: 1200 },
  { img: 'https://images.unsplash.com/photo-1576091160550-2173dba999ef?q=80&w=1000&h=700&auto=format&fit=crop',  year: '2024', width: 1000, height: 700  },
  { img: 'https://images.unsplash.com/photo-1507842217343-583bb7270b66?q=80&w=1000&h=700&auto=format&fit=crop',  year: '2026', width: 1000, height: 700  },
  { img: 'https://images.unsplash.com/photo-1536440136628-849c177e76a1?q=80&w=800&h=1200&auto=format&fit=crop',  year: '2025', width: 800,  height: 1200 },
];

// ── i18n computed ─────────────────────────────────────────────────────────────
const processSteps = computed(
  () => tm('process.steps') as Array<{ title: string; description: string }>
);
const processTags = computed(
  () => tm('process.tags') as Array<string[]>
);

// ── Active nav section (#5) ──────────────────────────────────────────────────
const activeSection = ref<string>('home');

function updateActiveSection() {
  const ids  = ['home', 'process', 'gallery', 'contact'];
  const mid  = window.scrollY + window.innerHeight * 0.4;
  for (let i = ids.length - 1; i >= 0; i--) {
    const el = document.getElementById(ids[i]);
    if (el && el.offsetTop <= mid) { activeSection.value = ids[i]; return; }
  }
  activeSection.value = 'home';
}

// ── Internal vars ─────────────────────────────────────────────────────────────
let ctx: gsap.Context;
let lenis: Lenis | null = null;
let cursorMoveHandler: ((e: MouseEvent) => void) | null = null;
let lenisTickerFn:    ((time: number) => void) | null = null;
let escHandler:       ((e: KeyboardEvent) => void) | null = null;

// ── Helpers ───────────────────────────────────────────────────────────────────
const scrollTo = (target: string) => {
  closeMenu();            // restarts Lenis before scroll (#3)
  lenis?.scrollTo(target, { duration: 1 });
};

const scrollToProcessCard = (index: number) => {
  if (!processSection.value || !processCardsWrapper.value || !lenis) return;
  const total    = processCardsWrapper.value.scrollWidth - window.innerWidth;
  const fraction = processSteps.value.length > 1 ? index / (processSteps.value.length - 1) : 0;
  lenis.scrollTo(processSection.value.offsetTop + fraction * total, { duration: 0.8 });
};

// ── Mount ─────────────────────────────────────────────────────────────────────
onMounted(() => {
  const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;

  // Preloader: skip entirely for reduced-motion users (#8)
  if (prefersReducedMotion) {
    preloaderDone.value  = true;
    heroStats.clients    = 80;
    heroStats.projects   = 150;
  } else {
    setTimeout(() => { preloaderDone.value = true; }, 950);
  }

  // Escape closes mobile menu
  escHandler = (e: KeyboardEvent) => { if (e.key === 'Escape') closeMenu(); };
  window.addEventListener('keydown', escHandler);

  // Lenis smooth scroll — renamed easing param to avoid shadowing i18n `t` (#11)
  lenis = new Lenis({
    duration: 1.2,
    easing: (progress) => Math.min(1, 1.001 - Math.pow(2, -10 * progress)),
    orientation: 'vertical',
    gestureOrientation: 'vertical',
    smoothWheel: true,
    wheelMultiplier: 1,
    touchMultiplier: 2,
    infinite: false,
  });

  // Active section tracking via Lenis scroll event (#5)
  lenis.on('scroll', () => {
    ScrollTrigger.update();
    updateActiveSection();
  });
  updateActiveSection(); // set initial state

  lenisTickerFn = (time) => { lenis?.raf(time * 1000); };
  gsap.ticker.add(lenisTickerFn);
  gsap.ticker.lagSmoothing(0);

  if (mainContainer.value) {
    ctx = gsap.context(() => {

      // ── Animations — only without reduced motion (#8) ──────────────────────
      if (!prefersReducedMotion) {
        const tl = gsap.timeline();

        // Hero ambient blobs — slow organic drift
        if (heroSection.value) {
          const blobs = heroSection.value.querySelectorAll('.hero__blob');
          blobs.forEach((blob, i) => {
            gsap.to(blob, {
              x: `${(i % 2 === 0 ? 1 : -1) * gsap.utils.random(40, 80)}`,
              y: `${gsap.utils.random(30, 70)}`,
              duration: gsap.utils.random(14, 22),
              ease: 'sine.inOut',
              repeat: -1,
              yoyo: true,
              delay: i * 2.2,
            });
          });

          // Spotlight — follows cursor inside hero
          const spotlight = heroSection.value.querySelector('.hero__spotlight') as HTMLElement | null;
          if (spotlight) {
            heroSection.value.addEventListener('mousemove', (e: MouseEvent) => {
              const rect = heroSection.value!.getBoundingClientRect();
              const x = ((e.clientX - rect.left) / rect.width) * 100;
              const y = ((e.clientY - rect.top) / rect.height) * 100;
              heroSection.value!.style.setProperty('--mouse-x', `${x}%`);
              heroSection.value!.style.setProperty('--mouse-y', `${y}%`);
            });
          }
        }

        // Hero title char reveal + descriptor + CTAs + stats + counters
        if (heroTitle.value) {
          const letters = heroTitle.value.querySelectorAll('.char');
          tl.from(letters, { y: '110%', opacity: 0, duration: 1.3, stagger: 0.03, ease: 'power4.out', delay: 0.1 })
            .from('.hero__eyebrow', { y: 12, autoAlpha: 0, duration: 0.7, ease: 'power3.out' }, 0.3)
            .from('.hero__descriptor', { y: 16, autoAlpha: 0, duration: 0.8, ease: 'power3.out' }, '-=0.5')
            .from('.hero__ticker', { autoAlpha: 0, duration: 0.6, ease: 'power2.out' }, '-=0.4');

          if (heroBtns.value) {
            tl.fromTo(heroBtns.value.querySelectorAll('.hero__cta-primary, .hero__cta-ghost'),
              { y: 18, autoAlpha: 0 },
              { y: 0, autoAlpha: 1, duration: 0.8, stagger: 0.14, ease: 'power3.out', clearProps: 'all' },
              '-=0.35'
            );
          }

          tl.from('.hero__bottom', { autoAlpha: 0, y: 14, duration: 1, ease: 'power3.out' }, '-=0.4')
            .to(heroStats, { clients: 80, projects: 150, duration: 2, ease: 'power2.out', snap: { clients: 1, projects: 1 } }, '<0.2');
        }

        // Navbar entrance
        if (navbar.value) {
          gsap.fromTo(navbar.value,
            { y: -20, autoAlpha: 0 },
            { y: 0, autoAlpha: 1, duration: 1, ease: 'power3.out', delay: 0.8, clearProps: 'all' }
          );
        }

        // Gallery curtain wipe reveal
        if (galleryImages.value.length > 0) {
          ScrollTrigger.batch(galleryImages.value, {
            onEnter: (elements) => {
              gsap.fromTo(elements,
                { clipPath: 'inset(100% 0% 0% 0% round 12px)' },
                { clipPath: 'inset(0% 0% 0% 0% round 12px)', duration: 1.4, stagger: 0.18, ease: 'power3.out', overwrite: true }
              );
            },
            start: 'top 85%',
          });
        }

        // Global text reveals
        mainContainer.value?.querySelectorAll('.reveal-text').forEach((el) => {
          gsap.fromTo(el,
            { autoAlpha: 0, y: -20 },
            { autoAlpha: 1, y: 0, duration: 1, ease: 'power3.out',
              scrollTrigger: { trigger: el, start: 'top 60%', toggleActions: 'play none none reverse' } }
          );
        });

      } else {
        // ── Reduced-motion fallback: show everything instantly (#8) ──────────
        mainContainer.value?.querySelectorAll('.reveal-text').forEach((el) => {
          gsap.set(el, { autoAlpha: 1, y: 0, clearProps: 'all' });
        });
        if (heroTitle.value) {
          heroTitle.value.querySelectorAll('.char').forEach((el) => {
            gsap.set(el, { y: 0, opacity: 1, clearProps: 'all' });
          });
        }
        galleryImages.value.forEach((el) => {
          gsap.set(el, { clipPath: 'none' });
        });
      }

      // ── Always: navbar scrolled class ───────────────────────────────────────
      if (navbar.value) {
        ScrollTrigger.create({
          start: 'top -50',
          onEnter:     () => navbar.value?.classList.add('navbar--scrolled'),
          onLeaveBack: () => navbar.value?.classList.remove('navbar--scrolled'),
        });
      }

      // ── Always: process horizontal scroll pin ────────────────────────────────
      // Uses function-based values + invalidateOnRefresh to survive window resize (#1)
      if (processCardsWrapper.value && processSection.value) {
        const getTotal = () => processCardsWrapper.value!.scrollWidth - window.innerWidth;
        gsap.to(processCardsWrapper.value, {
          x: () => -getTotal(),
          ease: 'none',
          scrollTrigger: {
            trigger: processSection.value,
            pin: true,
            scrub: 1,
            start: 'top top',
            end: () => `+=${getTotal()}`,
            invalidateOnRefresh: true,
          },
        });
      }

    }, mainContainer.value);
  }

  // Custom cursor (pointer devices only)
  if (cursor.value && window.matchMedia('(pointer: fine)').matches) {
    gsap.set(cursor.value, { x: -100, y: -100 });
    cursorMoveHandler = (e: MouseEvent) => {
      gsap.to(cursor.value, { x: e.clientX, y: e.clientY, duration: 0.12, ease: 'power3.out' });
    };
    window.addEventListener('mousemove', cursorMoveHandler);

    document.querySelectorAll('a, button, .gallery__item-inner, .project-card, .budget-pill, .lang-btn, .nav-link').forEach(el => {
      el.addEventListener('mouseenter', () => cursor.value?.classList.add('is-hovering'));
      el.addEventListener('mouseleave', () => cursor.value?.classList.remove('is-hovering'));
    });
  }
});

onUnmounted(() => {
  if (ctx)             ctx.revert();
  if (lenis)           { lenis.destroy(); lenis = null; }
  if (cursorMoveHandler) window.removeEventListener('mousemove', cursorMoveHandler);
  if (lenisTickerFn)   { gsap.ticker.remove(lenisTickerFn); lenisTickerFn = null; }
  if (escHandler)      { window.removeEventListener('keydown', escHandler); escHandler = null; }
});
</script>

<template>
  <!-- Preloader -->
  <Transition name="preloader-fade">
    <div v-if="!preloaderDone" class="preloader" aria-hidden="true">
      <div class="preloader__ring-wrap">
        <svg class="preloader__ring" viewBox="0 0 100 100" aria-hidden="true">
          <defs>
            <linearGradient id="plGradient" x1="0%" y1="0%" x2="100%" y2="100%">
              <stop offset="0%"   stop-color="#00ffcc"/>
              <stop offset="100%" stop-color="#ff00cc"/>
            </linearGradient>
          </defs>
          <circle class="preloader__ring-track" cx="50" cy="50" r="38"/>
          <circle class="preloader__ring-fill"  cx="50" cy="50" r="38"/>
        </svg>
        <span class="preloader__mark">VS</span>
      </div>
      <p class="preloader__name">V-STUDIOS</p>
      <p class="preloader__sub">Creative Studio</p>
    </div>
  </Transition>

  <div ref="cursor" class="cursor" aria-hidden="true"></div>

  <!-- Mobile menu (#2) -->
  <Transition name="mobile-menu">
    <div v-if="menuOpen" class="mobile-menu" role="dialog" aria-modal="true" :aria-label="t('nav.open_menu')">
      <div class="mobile-menu__backdrop" @click="closeMenu" aria-hidden="true"></div>
      <nav class="mobile-menu__content">
        <button class="mobile-menu__close" @click="closeMenu" :aria-label="t('nav.close_menu')">✕</button>
        <a href="#process" class="mobile-menu__link" :class="{ 'is-active': activeSection === 'process' }" @click.prevent="scrollTo('#process')">{{ t('nav.approach') }}</a>
        <a href="#gallery"  class="mobile-menu__link" :class="{ 'is-active': activeSection === 'gallery'  }" @click.prevent="scrollTo('#gallery')">{{ t('nav.gallery') }}</a>
        <a href="#contact"  class="mobile-menu__link" :class="{ 'is-active': activeSection === 'contact'  }" @click.prevent="scrollTo('#contact')">{{ t('nav.project') }}</a>
        <div class="mobile-menu__lang">
          <button
            v-for="lang in ['fr', 'en', 'es']"
            :key="lang"
            class="lang-btn"
            :class="{ 'lang-btn--active': locale === lang }"
            :aria-pressed="locale === lang"
            :aria-label="lang === 'fr' ? 'Français' : lang === 'en' ? 'English' : 'Español'"
            @click="locale = lang"
          >{{ lang.toUpperCase() }}</button>
        </div>
      </nav>
    </div>
  </Transition>

  <main ref="mainContainer" class="landing-page dark-mode">
    <!-- NAVIGATION -->
    <nav ref="navbar" class="navbar">
      <div class="navbar__container">
        <a href="#" class="navbar__logo">
          <span class="logo-dot"></span>
          V-STUDIOS DEMO
        </a>

        <div class="navbar__menu">
          <!-- Active state on nav links (#5) -->
          <a href="#process" class="nav-link" :class="{ 'is-active': activeSection === 'process' }" @click.prevent="scrollTo('#process')">{{ t('nav.approach') }}</a>
          <a href="#gallery"  class="nav-link" :class="{ 'is-active': activeSection === 'gallery'  }" @click.prevent="scrollTo('#gallery')">{{ t('nav.gallery') }}</a>
          <a href="#contact"  class="nav-link" :class="{ 'is-active': activeSection === 'contact'  }" @click.prevent="scrollTo('#contact')">{{ t('nav.project') }}</a>
        </div>

        <div class="navbar__right">
          <div class="lang-switcher">
            <button
              v-for="lang in ['fr', 'en', 'es']"
              :key="lang"
              class="lang-btn"
              :class="{ 'lang-btn--active': locale === lang }"
              :aria-pressed="locale === lang"
              :aria-label="lang === 'fr' ? 'Français' : lang === 'en' ? 'English' : 'Español'"
              @click="locale = lang"
            >{{ lang.toUpperCase() }}</button>
          </div>
          <button class="navbar__cta" @click="scrollTo('#contact')">
            <span class="cta-text">{{ t('nav.contact') }}</span>
            <span class="cta-arrow">↗</span>
          </button>
          <!-- Hamburger — 3 bars + animates to × when open (#4) -->
          <button
            class="navbar__hamburger"
            :class="{ 'is-open': menuOpen }"
            @click="toggleMenu"
            :aria-label="t('nav.open_menu')"
            :aria-expanded="menuOpen"
          >
            <span></span>
            <span></span>
            <span></span>
          </button>
        </div>
      </div>
    </nav>

    <!-- HERO SECTION -->
    <section ref="heroSection" class="hero" id="home" aria-label="Hero">
      <!-- Layered ambient background -->
      <div class="hero__canvas" aria-hidden="true">
        <div class="hero__blob hero__blob--cyan"></div>
        <div class="hero__blob hero__blob--violet"></div>
        <div class="hero__blob hero__blob--rose"></div>
      </div>
      <div class="hero__spotlight" aria-hidden="true"></div>
      <div class="hero__grid" aria-hidden="true"></div>
      <div class="hero__vignette" aria-hidden="true"></div>
      <div class="hero__grain" aria-hidden="true"></div>

      <!-- Main stage -->
      <div class="hero__stage">
        <div class="hero__eyebrow" aria-hidden="true">
          <span class="hero__eyebrow-tag">Creative Studio</span>
          <span class="hero__eyebrow-sep">·</span>
          <span>Est. 2019</span>
        </div>

        <h1 ref="heroTitle" class="hero__title">
          <span class="hero__word" v-for="(chars, wi) in heroWords" :key="wi">
            <span class="char" v-for="(char, ci) in chars" :key="`${wi}-${ci}`">{{ char }}</span>
          </span>
        </h1>

        <p class="hero__descriptor">{{ t('hero.subtitle') }}</p>

        <div class="hero__ticker" aria-hidden="true">
          <div class="hero__ticker-track">
            <span v-for="n in 4" :key="n">Design · Motion · Vue · GSAP · Performance · Branding ·&nbsp;</span>
          </div>
        </div>

        <div ref="heroBtns" class="hero__actions">
          <button class="hero__cta-primary" @click="scrollTo('#contact')">
            <span>{{ t('hero.cta_primary') }}</span>
            <span class="hero__cta-icon" aria-hidden="true">↗</span>
          </button>
          <button class="hero__cta-ghost" @click="scrollTo('#gallery')">
            {{ t('hero.cta_secondary') }}
            <span class="hero__cta-line"></span>
          </button>
        </div>
      </div>

      <!-- Bottom strip: stats + scroll badge -->
      <div class="hero__bottom">
        <div class="hero__stats">
          <div class="hero__stat">
            <span class="hero__stat-num">{{ heroStats.clients }}<sup>+</sup></span>
            <span class="hero__stat-lbl">{{ t('hero.stat_clients_label') }}</span>
          </div>
          <div class="hero__stat-sep" aria-hidden="true"></div>
          <div class="hero__stat">
            <span class="hero__stat-num">{{ heroStats.projects }}<sup>+</sup></span>
            <span class="hero__stat-lbl">{{ t('hero.stat_projects_label') }}</span>
          </div>
          <div class="hero__stat-sep" aria-hidden="true"></div>
          <div class="hero__stat">
            <span class="hero__stat-num">{{ t('hero.stat_since_value') }}</span>
            <span class="hero__stat-lbl">{{ t('hero.stat_since_label') }}</span>
          </div>
        </div>
        <div class="hero__scroll-badge" aria-hidden="true">
          <span class="hero__scroll-ring">
            <span class="hero__scroll-bead"></span>
          </span>
          <span class="hero__scroll-text">Scroll</span>
        </div>
      </div>
    </section>

    <!-- PROCESS SECTION (#9 aria-label) -->
    <section ref="processSection" class="process" id="process" :aria-label="t('process.title')">
      <div class="process__header">
        <div class="process__header-left">
          <span class="process__eyebrow">{{ t('process.eyebrow') }}</span>
          <h2 class="reveal-text">{{ t('process.title') }}</h2>
        </div>
        <div class="process__header-right">
          <span class="process__count reveal-text">{{ processSteps.length }} {{ t('process.steps_label') }}</span>
          <span class="process__scroll-hint">{{ t('process.scroll_hint') }}</span>
        </div>
      </div>
      <div ref="processCardsWrapper" class="process__cards" role="list">
        <div
          class="process-card"
          v-for="(step, i) in processSteps"
          :key="i"
          role="listitem"
          tabindex="0"
          @focus="scrollToProcessCard(i)"
        >
          <div class="process-card__inner">
            <div class="process-card__top">
              <span class="process-card__badge">0{{ i + 1 }}</span>
            </div>
            <div class="process-card__body">
              <h3 class="reveal-text">{{ step.title }}</h3>
              <p class="reveal-text">{{ step.description }}</p>
            </div>
            <div class="process-card__footer">
              <span v-for="tag in processTags[i]" :key="tag" class="process-card__tag">{{ tag }}</span>
            </div>
            <span class="process-card__watermark" aria-hidden="true">0{{ i + 1 }}</span>
          </div>
        </div>
      </div>
    </section>

    <!-- GALLERY SECTION (#9 aria-label, #7 clickable items) -->
    <section class="gallery" id="gallery" :aria-label="t('gallery.title_main')">
      <div class="gallery__header">
        <div class="gallery__header-left">
          <span class="gallery__eyebrow">{{ t('gallery.eyebrow') }}</span>
          <h2 class="reveal-text">{{ t('gallery.title_main') }} <em>{{ t('gallery.title_accent') }}</em></h2>
        </div>
        <div class="gallery__header-right">
          <span class="gallery__count reveal-text">{{ t('gallery.count', { count: galleryData.length }) }}</span>
          <a href="#contact" class="gallery__view-all reveal-text" @click.prevent="scrollTo('#contact')">
            {{ t('gallery.view_all') }} <span class="arrow">↗</span>
          </a>
        </div>
      </div>

      <div class="gallery__grid">
        <div
          v-for="(item, index) in galleryData"
          :key="index"
          class="gallery__item"
          :class="`gallery__item--${index}`"
          ref="galleryImages"
        >
          <!-- Gallery items are now clickable → contact (#7) -->
          <div
            class="gallery__item-inner"
            role="button"
            tabindex="0"
            :aria-label="`${t(`gallery.projects.${index}.title`)} — ${t('gallery.view_project')}`"
            @click="scrollTo('#contact')"
            @keydown.enter="scrollTo('#contact')"
            @keydown.space.prevent="scrollTo('#contact')"
          >
            <!-- First image eager, rest lazy (#3); dimensions for CLS (#4); decoding (#12) -->
            <img
              :src="item.img"
              :alt="t(`gallery.projects.${index}.title`)"
              :loading="index === 0 ? 'eager' : 'lazy'"
              :width="item.width"
              :height="item.height"
              decoding="async"
            />
            <div class="gallery__overlay">
              <div class="gallery__overlay-content">
                <span class="gallery__category">{{ t(`gallery.projects.${index}.category`) }}</span>
                <div class="gallery__meta">
                  <h3 class="gallery__item-title">{{ t(`gallery.projects.${index}.title`) }}</h3>
                  <span class="gallery__year">{{ item.year }}</span>
                </div>
              </div>
            </div>
            <span class="gallery__index" aria-hidden="true">0{{ index + 1 }}</span>
          </div>
        </div>
      </div>
    </section>

    <!-- CONTACT SECTION (#9 aria-label) -->
    <section class="contact" id="contact" :aria-label="t('contact.title')">
      <div class="contact__container">
        <div class="contact__header">
          <h2 class="reveal-text">{{ t('contact.title') }}</h2>
        </div>
        <ContactForm />
      </div>
    </section>

    <!-- FOOTER -->
    <footer class="footer">
      <div class="footer__container">
        <div class="footer__left">
          <p>{{ t('footer.copyright') }}</p>
        </div>
        <div class="footer__center">
          <p>{{ t('footer.tagline') }}</p>
        </div>
        <div class="footer__right">
          <div class="footer__links">
            <a href="#" class="footer__link--placeholder" aria-disabled="true" tabindex="-1">{{ t('footer.privacy') }}</a>
            <a href="#" class="footer__link--placeholder" aria-disabled="true" tabindex="-1">{{ t('footer.terms') }}</a>
          </div>
        </div>
      </div>
    </footer>
  </main>
</template>

<style lang="scss">
:root {
  --color-bg: #10082e;
  --color-text: #f0f0f0;
  --color-accent: #00ffcc;
  --color-accent-gradient: linear-gradient(135deg, #00ffcc 0%, #ff00cc 100%);
  --font-primary: 'Inter', 'Helvetica Neue', sans-serif;
  --transition-smooth: 0.4s cubic-bezier(0.25, 1, 0.5, 1);
  --section-padding: clamp(4rem, 10vh, 8rem) 5vw;
}

*,
*::before,
*::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

/* Global focus ring */
:focus-visible {
  outline: 2px solid var(--color-accent);
  outline-offset: 3px;
  border-radius: 4px;
}

body {
  background-color: var(--color-bg);
  color: var(--color-text);
  font-family: var(--font-primary);
  overflow-x: hidden;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;

  &.has-smooth-scrolling { overflow: hidden; }
}

html.lenis { height: auto; }
.lenis.lenis-smooth { scroll-behavior: auto; }
.lenis.lenis-smooth [data-lenis-prevent] { overscroll-behavior: contain; }
.lenis.lenis-stopped { overflow: hidden; }
.lenis.lenis-scrolling iframe { pointer-events: none; }

@mixin flex-center {
  display: flex;
  justify-content: center;
  align-items: center;
}

/* =========================================================================
   PRELOADER
   ========================================================================= */
.preloader {
  position: fixed;
  inset: 0;
  z-index: 100000;
  background: var(--color-bg);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 1.25rem;
  pointer-events: none;

  &::after {
    content: '';
    position: absolute;
    inset: 0;
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='240' height='240'%3E%3Cfilter id='n'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.8' numOctaves='4' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='240' height='240' filter='url(%23n)'/%3E%3C/svg%3E");
    opacity: 0.04;
    pointer-events: none;
  }
}

.preloader__ring-wrap {
  position: relative;
  width: 110px;
  height: 110px;
  display: flex;
  align-items: center;
  justify-content: center;
  animation: preloader-pop 0.55s cubic-bezier(0.34, 1.56, 0.64, 1) 0.05s both;
}

.preloader__ring {
  position: absolute;
  inset: 0;
  transform: rotate(-90deg);
  overflow: visible;
}

.preloader__ring-track {
  fill: none;
  stroke: rgba(255, 255, 255, 0.05);
  stroke-width: 1.2;
}

.preloader__ring-fill {
  fill: none;
  stroke: url(#plGradient);
  stroke-width: 1.5;
  stroke-linecap: round;
  stroke-dasharray: 238.76;
  stroke-dashoffset: 238.76;
  filter: drop-shadow(0 0 5px #00ffcc) drop-shadow(0 0 14px rgba(0, 255, 204, 0.45));
  animation: preloader-ring 0.75s cubic-bezier(0.4, 0, 0.2, 1) 0.15s forwards;
}

@keyframes preloader-ring {
  to { stroke-dashoffset: 0; }
}

.preloader__mark {
  position: relative;
  z-index: 1;
  font-family: var(--font-primary);
  font-size: 1.15rem;
  font-weight: 900;
  letter-spacing: -0.03em;
  background: linear-gradient(135deg, #fff 40%, rgba(0, 255, 204, 0.85) 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  animation: preloader-mark 0.45s cubic-bezier(0.34, 1.56, 0.64, 1) 0.4s both;
}

@keyframes preloader-mark {
  from { opacity: 0; transform: scale(0.5); }
  to   { opacity: 1; transform: scale(1); }
}

.preloader__name {
  font-size: 0.62rem;
  font-weight: 800;
  letter-spacing: 0.55em;
  text-transform: uppercase;
  color: rgba(255, 255, 255, 0.6);
  animation: preloader-rise 0.5s ease 0.35s both;
  margin-right: -0.55em;
}

.preloader__sub {
  font-size: 0.5rem;
  font-weight: 400;
  letter-spacing: 0.35em;
  text-transform: uppercase;
  color: rgba(255, 255, 255, 0.18);
  animation: preloader-rise 0.5s ease 0.45s both;
  margin-right: -0.35em;
}

@keyframes preloader-pop {
  from { opacity: 0; transform: scale(0.85); }
  to   { opacity: 1; transform: scale(1); }
}

@keyframes preloader-rise {
  from { opacity: 0; transform: translateY(7px); }
  to   { opacity: 1; transform: translateY(0); }
}

.preloader-fade-leave-active {
  transition: opacity 0.6s ease, transform 0.6s cubic-bezier(0.4, 0, 0.2, 1);
}
.preloader-fade-leave-to { opacity: 0; transform: scale(1.03); }

/* =========================================================================
   PREFERS-REDUCED-MOTION (#8)
   ========================================================================= */
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }

  .preloader { display: none; }

  .char {
    transform: none !important;
    opacity: 1 !important;
  }

  .reveal-text {
    visibility: visible !important;
    opacity: 1 !important;
  }

  .gallery__item {
    clip-path: none !important;
  }
}

/* =========================================================================
   NAVIGATION
   ========================================================================= */
@keyframes pulse-dot {
  0%, 100% { opacity: 1;    box-shadow: 0 0 6px var(--color-accent), 0 0 12px rgba(0,255,204,0.4); }
  50%       { opacity: 0.35; box-shadow: 0 0 3px var(--color-accent); }
}

.navbar {
  position: fixed;
  top: 1.5rem;
  left: 50%;
  transform: translateX(-50%);
  width: 90%;
  max-width: 1200px;
  z-index: 100;
  padding: 0.6rem 0.6rem 0.6rem 1.5rem;
  background: rgba(8, 8, 8, 0.65);
  backdrop-filter: blur(28px) saturate(180%);
  -webkit-backdrop-filter: blur(28px) saturate(180%);
  border: 1px solid rgba(255, 255, 255, 0.07);
  border-radius: 100px;
  box-shadow: inset 0 1px 0 rgba(255,255,255,0.06), 0 8px 32px rgba(0,0,0,0.5);
  transition: all 0.5s cubic-bezier(0.25, 1, 0.5, 1);
  will-change: transform, width, box-shadow;

  &--scrolled {
    padding: 0.5rem 0.5rem 0.5rem 1.25rem;
    background: rgba(4, 4, 4, 0.85);
    width: 84%;
    border-color: rgba(0, 255, 204, 0.14);
    box-shadow: inset 0 1px 0 rgba(0,255,204,0.06), 0 16px 48px rgba(0,0,0,0.7), 0 0 60px rgba(0,255,204,0.03);
  }

  &__container {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 1.5rem;
  }

  &__logo {
    display: flex;
    align-items: center;
    gap: 0.55rem;
    font-size: 0.95rem;
    font-weight: 800;
    letter-spacing: 0.14em;
    color: #fff;
    text-decoration: none;
    flex-shrink: 0;
    transition: opacity 0.3s ease;

    .logo-dot {
      width: 7px;
      height: 7px;
      border-radius: 50%;
      background: var(--color-accent);
      flex-shrink: 0;
      animation: pulse-dot 2.8s ease-in-out infinite;
    }

    &:hover { opacity: 0.65; }
  }

  &__menu {
    display: flex;
    gap: 0.15rem;
    flex: 1;
    justify-content: center;

    @media (max-width: 768px) { display: none; }

    .nav-link {
      position: relative;
      padding: 0.45rem 1rem 0.55rem;
      font-size: 0.68rem;
      font-weight: 600;
      text-transform: uppercase;
      letter-spacing: 0.18em;
      color: rgba(255, 255, 255, 0.55);
      text-decoration: none;
      border-radius: 100px;
      transition: color 0.25s ease, background 0.25s ease;

      &::after {
        content: '';
        position: absolute;
        bottom: 6px;
        left: 50%;
        transform: translateX(-50%) scale(0);
        width: 3px;
        height: 3px;
        border-radius: 50%;
        background: var(--color-accent);
        box-shadow: 0 0 6px var(--color-accent);
        transition: transform 0.3s cubic-bezier(0.34, 1.56, 0.64, 1);
      }

      &:hover {
        color: #fff;
        background: rgba(255, 255, 255, 0.05);
        &::after { transform: translateX(-50%) scale(1); }
      }

      /* Active section indicator (#5) */
      &.is-active {
        color: #fff;
        &::after { transform: translateX(-50%) scale(1); }
      }
    }
  }

  &__right {
    display: flex;
    align-items: center;
    gap: 0.6rem;
    flex-shrink: 0;
  }

  &__cta {
    display: flex;
    align-items: center;
    gap: 0.45rem;
    background: var(--color-accent);
    color: #000;
    border: none;
    padding: 0.6rem 1.2rem;
    border-radius: 100px;
    font-family: inherit;
    font-size: 0.68rem;
    font-weight: 800;
    text-transform: uppercase;
    letter-spacing: 0.08em;
    cursor: pointer;
    transition: all 0.3s cubic-bezier(0.25, 1, 0.5, 1);
    white-space: nowrap;

    .cta-text  { display: inline; }
    .cta-arrow {
      display: inline-block;
      font-size: 0.8rem;
      line-height: 1;
      transition: transform 0.3s cubic-bezier(0.34, 1.56, 0.64, 1);
    }

    &:hover {
      transform: translateY(-2px);
      box-shadow: 0 0 0 3px rgba(0,255,204,0.15), 0 8px 24px rgba(0,255,204,0.3);
      .cta-arrow { transform: translate(2px, -2px); }
    }

    &:active { transform: translateY(0); }
  }

  /* Hamburger — 3 bars, animates to × (#4) */
  &__hamburger {
    display: none;
    flex-direction: column;
    justify-content: center;
    gap: 5px;
    width: 34px;
    height: 34px;
    background: transparent;
    border: none;
    cursor: pointer;
    padding: 6px;
    border-radius: 8px;
    transition: background 0.2s ease;

    span {
      display: block;
      width: 100%;
      height: 1.5px;
      background: rgba(255, 255, 255, 0.65);
      border-radius: 1px;
      transition:
        transform  0.35s cubic-bezier(0.25, 1, 0.5, 1),
        opacity    0.2s ease,
        background 0.2s ease;
    }

    &:hover {
      background: rgba(255, 255, 255, 0.06);
      span { background: #fff; }
    }

    /* × state */
    &.is-open {
      span:nth-child(1) { transform: translateY(6.5px) rotate(45deg); }
      span:nth-child(2) { opacity: 0; transform: scaleX(0); }
      span:nth-child(3) { transform: translateY(-6.5px) rotate(-45deg); }
    }

    @media (max-width: 768px) { display: flex; }
  }

  @media (max-width: 600px) {
    top: 1rem;
    width: 94%;
    padding: 0.45rem 0.45rem 0.45rem 1.1rem;

    &--scrolled { width: 90%; }

    &__logo {
      font-size: 0.82rem;
      letter-spacing: 0.1em;
      .logo-dot { width: 6px; height: 6px; }
    }

    &__right { gap: 0.4rem; }

    &__cta {
      padding: 0.58rem 0.65rem;
      gap: 0;
      .cta-text  { display: none; }
      .cta-arrow { font-size: 0.9rem; }
    }
  }
}

.lang-switcher {
  display: flex;
  align-items: center;
  background: rgba(255, 255, 255, 0.04);
  border: 1px solid rgba(255, 255, 255, 0.07);
  border-radius: 100px;
  padding: 0.2rem;
  gap: 0.1rem;

  .lang-btn {
    background: transparent;
    border: none;
    padding: 0.3rem 0.55rem;
    font-family: inherit;
    font-size: 0.6rem;
    font-weight: 700;
    letter-spacing: 0.12em;
    color: rgba(255, 255, 255, 0.4);
    cursor: pointer;
    border-radius: 100px;
    transition: all 0.2s ease;
    line-height: 1;

    &:hover:not(.lang-btn--active) { color: rgba(255, 255, 255, 0.6); background: rgba(255, 255, 255, 0.05); }

    &--active {
      background: rgba(0, 255, 204, 0.1);
      color: var(--color-accent);
      box-shadow: 0 0 10px rgba(0, 255, 204, 0.12);
    }
  }

  @media (max-width: 600px) {
    padding: 0.15rem;
    .lang-btn { padding: 0.28rem 0.42rem; font-size: 0.56rem; letter-spacing: 0.08em; }
  }
}

/* =========================================================================
   MOBILE MENU
   ========================================================================= */
.mobile-menu {
  position: fixed;
  inset: 0;
  z-index: 9999;
  display: flex;

  &__backdrop {
    position: absolute;
    inset: 0;
    background: rgba(0, 0, 0, 0.65);
    backdrop-filter: blur(6px);
    -webkit-backdrop-filter: blur(6px);
  }

  &__content {
    position: relative;
    margin-left: auto;
    width: min(320px, 85vw);
    height: 100%;
    background: #0d0820;
    border-left: 1px solid rgba(255, 255, 255, 0.07);
    display: flex;
    flex-direction: column;
    padding: 1.5rem 2rem 2.5rem;
    gap: 0.25rem;
    overflow-y: auto;
  }

  &__close {
    align-self: flex-end;
    background: transparent;
    border: none;
    color: rgba(255, 255, 255, 0.45);
    font-size: 1.1rem;
    cursor: pointer;
    padding: 0.5rem;
    margin-bottom: 1.75rem;
    border-radius: 8px;
    transition: color 0.2s ease, background 0.2s ease;
    line-height: 1;
    &:hover { color: #fff; background: rgba(255, 255, 255, 0.06); }
  }

  &__link {
    font-size: 2.2rem;
    font-weight: 800;
    letter-spacing: -0.02em;
    color: rgba(255, 255, 255, 0.55);
    text-decoration: none;
    line-height: 1.2;
    padding: 0.6rem 0;
    transition: color 0.25s ease;
    border-bottom: 1px solid rgba(255, 255, 255, 0.05);

    &:hover   { color: var(--color-accent); }
    &.is-active { color: var(--color-accent); } /* Active section in mobile (#5) */
  }

  &__lang {
    display: flex;
    gap: 0.5rem;
    margin-top: auto;
    padding-top: 2.5rem;
  }
}

.mobile-menu-enter-active { transition: opacity 0.3s ease; }
.mobile-menu-leave-active  { transition: opacity 0.25s ease; }
.mobile-menu-enter-from,
.mobile-menu-leave-to      { opacity: 0; }

.mobile-menu-enter-active .mobile-menu__content { transition: transform 0.35s cubic-bezier(0.25, 1, 0.5, 1); }
.mobile-menu-enter-from   .mobile-menu__content { transform: translateX(100%); }
.mobile-menu-leave-active .mobile-menu__content { transition: transform 0.25s ease; }
.mobile-menu-leave-to     .mobile-menu__content { transform: translateX(100%); }

/* =========================================================================
   CUSTOM CURSOR
   ========================================================================= */
.cursor {
  position: fixed;
  top: 0;
  left: 0;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  border: 1.5px solid rgba(0, 255, 204, 0.65);
  pointer-events: none;
  z-index: 99999;
  transform: translate(-50%, -50%);
  will-change: transform;
  transition:
    width 0.35s cubic-bezier(0.25, 1, 0.5, 1),
    height 0.35s cubic-bezier(0.25, 1, 0.5, 1),
    background 0.35s ease,
    border-color 0.35s ease;

  &.is-hovering {
    width: 46px;
    height: 46px;
    background: rgba(0, 255, 204, 0.07);
    border-color: rgba(0, 255, 204, 0.35);
  }

  @media (pointer: coarse) { display: none; }
}

.landing-page,
.landing-page * { cursor: none; }

.landing-page input,
.landing-page textarea { cursor: text; }

.landing-page *:focus-visible { cursor: auto; }

.landing-page { width: 100%; }

.reveal-text {
  visibility: hidden;
  opacity: 0;
  will-change: transform, opacity;
}

/* =========================================================================
   HERO SECTION — IMMERSIVE REDESIGN
   ========================================================================= */
.hero {
  position: relative;
  height: 100svh;
  width: 100vw;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background: #060412;
  --mouse-x: 50%;
  --mouse-y: 50%;

  // ── Ambient canvas (blobs) ────────────────────────────────────────────────
  &__canvas {
    position: absolute;
    inset: 0;
    z-index: 0;
    overflow: hidden;
  }

  &__blob {
    position: absolute;
    border-radius: 50%;
    filter: blur(110px);
    will-change: transform;
    pointer-events: none;

    &--cyan {
      width: 60vw; height: 60vw;
      top: -18%; left: -12%;
      background: radial-gradient(circle, rgba(0, 255, 204, 0.22) 0%, transparent 68%);
    }

    &--violet {
      width: 48vw; height: 48vw;
      top: -8%; right: -8%;
      background: radial-gradient(circle, rgba(100, 30, 255, 0.30) 0%, transparent 68%);
    }

    &--rose {
      width: 44vw; height: 44vw;
      bottom: 2%; left: 20%;
      background: radial-gradient(circle, rgba(220, 20, 150, 0.22) 0%, transparent 68%);
    }
  }

  // ── Mouse-tracking spotlight ──────────────────────────────────────────────
  &__spotlight {
    position: absolute;
    inset: 0;
    z-index: 1;
    background: radial-gradient(
      700px circle at var(--mouse-x) var(--mouse-y),
      rgba(0, 255, 204, 0.055) 0%,
      transparent 55%
    );
    pointer-events: none;
    transition: background 0.08s linear;
  }

  // ── Perspective grid ──────────────────────────────────────────────────────
  &__grid {
    position: absolute;
    inset: 0;
    z-index: 0;
    background-image:
      linear-gradient(rgba(255, 255, 255, 0.022) 1px, transparent 1px),
      linear-gradient(90deg, rgba(255, 255, 255, 0.022) 1px, transparent 1px);
    background-size: 80px 80px;
    mask-image: radial-gradient(ellipse 80% 65% at 50% 50%, rgba(0,0,0,0.7), transparent);
    -webkit-mask-image: radial-gradient(ellipse 80% 65% at 50% 50%, rgba(0,0,0,0.7), transparent);
    pointer-events: none;
  }

  // ── Edge vignette + grain ─────────────────────────────────────────────────
  &__vignette {
    position: absolute;
    inset: 0;
    z-index: 2;
    background: radial-gradient(ellipse 110% 110% at 50% 50%, transparent 28%, #060412 100%);
    pointer-events: none;
  }

  &__grain {
    position: absolute;
    inset: 0;
    z-index: 3;
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='300' height='300'%3E%3Cfilter id='n'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.82' numOctaves='4' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='300' height='300' filter='url(%23n)'/%3E%3C/svg%3E");
    opacity: 0.038;
    pointer-events: none;
  }

  // ── Main stage ────────────────────────────────────────────────────────────
  &__stage {
    position: relative;
    z-index: 10;
    text-align: center;
    padding: 0 clamp(1.5rem, 5vw, 5rem);
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 1.6rem;
    width: 100%;
  }

  // ── Eyebrow ───────────────────────────────────────────────────────────────
  &__eyebrow {
    display: flex;
    align-items: center;
    gap: 0.55rem;
    font-size: 0.65rem;
    font-weight: 600;
    letter-spacing: 0.32em;
    text-transform: uppercase;
    color: rgba(255, 255, 255, 0.28);

    &-tag { color: rgba(0, 255, 204, 0.75); }
    &-sep { opacity: 0.25; }
  }

  // ── Title ─────────────────────────────────────────────────────────────────
  &__title {
    font-size: clamp(4rem, 14vw, 14rem);
    font-weight: 900;
    text-transform: uppercase;
    letter-spacing: -0.05em;
    line-height: 0.88;
    display: flex;
    flex-wrap: wrap;
    justify-content: center;
    gap: 0.1em;
  }

  &__word {
    display: inline-flex;
    overflow: hidden;
  }

  .char {
    display: inline-block;
    will-change: transform, opacity;
    transform: translateY(110%);
    background: linear-gradient(
      162deg,
      #ffffff 0%,
      rgba(255, 255, 255, 0.78) 50%,
      rgba(190, 165, 255, 0.52) 100%
    );
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    background-clip: text;
  }

  // ── Descriptor ────────────────────────────────────────────────────────────
  &__descriptor {
    font-size: clamp(0.95rem, 1.5vw, 1.3rem);
    font-weight: 300;
    letter-spacing: 0.04em;
    color: rgba(255, 255, 255, 0.42);
    max-width: 460px;
    line-height: 1.65;
  }

  // ── Ticker / marquee ──────────────────────────────────────────────────────
  &__ticker {
    width: 100%;
    max-width: 640px;
    overflow: hidden;
    mask-image: linear-gradient(90deg, transparent 0%, #000 10%, #000 90%, transparent 100%);
    -webkit-mask-image: linear-gradient(90deg, transparent 0%, #000 10%, #000 90%, transparent 100%);
    height: 1.4em;
  }

  &__ticker-track {
    display: inline-flex;
    white-space: nowrap;
    animation: hero-ticker 20s linear infinite;

    span {
      font-size: 0.65rem;
      font-weight: 600;
      letter-spacing: 0.28em;
      text-transform: uppercase;
      color: rgba(255, 255, 255, 0.18);
    }
  }

  @keyframes hero-ticker {
    from { transform: translateX(0); }
    to   { transform: translateX(-50%); }
  }

  // ── CTAs ──────────────────────────────────────────────────────────────────
  &__actions {
    display: flex;
    align-items: center;
    gap: 2.2rem;
    margin-top: 0.4rem;

    @media (max-width: 520px) { flex-direction: column; gap: 1.25rem; }
  }

  &__cta-primary {
    position: relative;
    display: flex;
    align-items: center;
    gap: 0.6rem;
    padding: 1.05rem 2.2rem;
    background: #fff;
    color: #060412;
    border: none;
    border-radius: 100px;
    font-family: inherit;
    font-size: 0.85rem;
    font-weight: 700;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    cursor: pointer;
    transition: transform 0.35s cubic-bezier(0.25, 1, 0.5, 1), box-shadow 0.35s ease;
    overflow: hidden;

    &::before {
      content: '';
      position: absolute;
      inset: 0;
      background: linear-gradient(135deg, #00ffcc 0%, #a060ff 100%);
      opacity: 0;
      transition: opacity 0.35s ease;
      border-radius: inherit;
    }

    span, .hero__cta-icon { position: relative; z-index: 1; }

    .hero__cta-icon {
      display: inline-block;
      font-size: 0.9em;
      transition: transform 0.3s cubic-bezier(0.34, 1.56, 0.64, 1);
    }

    &:hover {
      transform: translateY(-3px);
      box-shadow: 0 0 0 1px rgba(255,255,255,0.08), 0 14px 40px rgba(0,255,204,0.22);
      &::before { opacity: 1; }
      .hero__cta-icon { transform: translate(3px, -3px); }
    }

    &:active { transform: translateY(0); }
  }

  &__cta-ghost {
    position: relative;
    display: inline-flex;
    align-items: center;
    background: transparent;
    border: none;
    font-family: inherit;
    font-size: 0.85rem;
    font-weight: 500;
    letter-spacing: 0.06em;
    color: rgba(255, 255, 255, 0.5);
    cursor: pointer;
    padding: 0.3rem 0;
    transition: color 0.25s ease;

    .hero__cta-line {
      position: absolute;
      bottom: 0;
      left: 0;
      width: 100%;
      height: 1px;
      background: rgba(255, 255, 255, 0.12);
      transition: background 0.25s ease;
    }

    &:hover {
      color: rgba(255, 255, 255, 0.88);
      .hero__cta-line { background: var(--color-accent); }
    }
  }

  // ── Bottom strip ──────────────────────────────────────────────────────────
  &__bottom {
    position: absolute;
    bottom: 0;
    left: 0;
    width: 100%;
    padding: 2rem 5vw 2.2rem;
    z-index: 20;
    display: flex;
    justify-content: space-between;
    align-items: flex-end;
    background: linear-gradient(to top, rgba(6, 4, 18, 0.92) 0%, rgba(6, 4, 18, 0.5) 55%, transparent 100%);
    border-top: 1px solid rgba(255, 255, 255, 0.035);

    @media (max-width: 600px) {
      flex-direction: column;
      align-items: flex-start;
      gap: 1.5rem;
      padding: 1.5rem 5vw 1.75rem;
    }
  }

  &__stats {
    display: flex;
    align-items: center;
    gap: clamp(1.5rem, 4vw, 3rem);
  }

  &__stat {
    display: flex;
    flex-direction: column;
    gap: 0.28rem;
  }

  &__stat-num {
    font-size: clamp(1.65rem, 3.2vw, 2.4rem);
    font-weight: 800;
    letter-spacing: -0.04em;
    color: #fff;
    line-height: 1;
    font-variant-numeric: tabular-nums;

    sup {
      font-size: 0.5em;
      font-weight: 600;
      opacity: 0.55;
      vertical-align: super;
    }
  }

  &__stat-lbl {
    font-size: 0.6rem;
    font-weight: 400;
    letter-spacing: 0.2em;
    text-transform: uppercase;
    color: rgba(255, 255, 255, 0.28);
  }

  &__stat-sep {
    width: 1px;
    height: 2.8rem;
    background: rgba(255, 255, 255, 0.07);
    flex-shrink: 0;
  }

  // ── Scroll badge (mouse / wheel) ──────────────────────────────────────────
  &__scroll-badge {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 0.55rem;

    @media (max-width: 600px) { display: none; }
  }

  &__scroll-ring {
    display: flex;
    justify-content: center;
    padding-top: 7px;
    width: 32px;
    height: 52px;
    border: 1px solid rgba(255, 255, 255, 0.1);
    border-radius: 16px;
  }

  &__scroll-bead {
    display: block;
    width: 4px;
    height: 10px;
    background: var(--color-accent);
    border-radius: 2px;
    box-shadow: 0 0 8px rgba(0, 255, 204, 0.65);
    animation: scroll-bead 2.5s cubic-bezier(0.45, 0, 0.55, 1) infinite;
  }

  @keyframes scroll-bead {
    0%   { transform: translateY(0);    opacity: 1; }
    65%  { transform: translateY(22px); opacity: 0.4; }
    100% { transform: translateY(0);    opacity: 1; }
  }

  &__scroll-text {
    font-size: 0.52rem;
    font-weight: 600;
    letter-spacing: 0.22em;
    text-transform: uppercase;
    color: rgba(255, 255, 255, 0.18);
  }
}

/* =========================================================================
   PROCESS SECTION
   ========================================================================= */
.process {
  min-height: 100vh;
  width: 100%;
  padding-top: 10vh;
  background-color: var(--color-bg);
  overflow: hidden;

  &__header {
    padding: 0 5vw;
    margin-bottom: 5vh;
    display: flex;
    justify-content: space-between;
    align-items: flex-end;

    &-left { display: flex; flex-direction: column; gap: 0.6rem; }
    &-right { display: flex; flex-direction: column; align-items: flex-end; gap: 0.4rem; padding-bottom: 0.2rem; }
  }

  &__eyebrow {
    font-size: 0.72rem;
    letter-spacing: 0.3em;
    text-transform: uppercase;
    color: var(--color-accent);
    font-weight: 500;
  }

  &__count {
    font-size: 0.68rem;
    letter-spacing: 0.2em;
    text-transform: uppercase;
    color: rgba(255, 255, 255, 0.22);
  }

  &__scroll-hint {
    font-size: 0.62rem;
    letter-spacing: 0.15em;
    text-transform: uppercase;
    color: rgba(255, 255, 255, 0.12);
  }

  h2 {
    font-size: clamp(2.5rem, 5vw, 4.5rem);
    font-weight: 800;
    letter-spacing: -0.03em;
    line-height: 1;
    color: #fff;
  }

  &__cards {
    display: flex;
    width: max-content;
    padding: 0 5vw;
    gap: clamp(1.5rem, 3vw, 2.5rem);
    will-change: transform;
  }

  &-card {
    width: clamp(340px, 58vw, 720px);
    height: 62vh;
    border-radius: 28px;
    background: linear-gradient(155deg, #1a0d3e 0%, #0e0630 100%);
    border: 1px solid rgba(255, 255, 255, 0.06);
    overflow: hidden;
    position: relative;
    box-shadow: 0 40px 80px rgba(0, 0, 0, 0.6);
    transition: transform 0.45s cubic-bezier(0.25, 1, 0.5, 1), box-shadow 0.45s cubic-bezier(0.25, 1, 0.5, 1);

    &::before {
      content: '';
      position: absolute;
      top: -80px;
      right: -80px;
      width: 260px;
      height: 260px;
      border-radius: 50%;
      filter: blur(80px);
      pointer-events: none;
      transition: opacity 0.4s ease;
    }

    &::after {
      content: '';
      position: absolute;
      top: 0; left: 0; right: 0;
      height: 1px;
      background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.12), transparent);
    }

    &:nth-child(1)::before { background: #00ffcc; opacity: 0.09; }
    &:nth-child(2)::before { background: #ff00cc; opacity: 0.07; }
    &:nth-child(3)::before { background: #7b2fff; opacity: 0.08; }

    &:hover {
      transform: translateY(-10px);
      box-shadow: 0 60px 100px rgba(0, 0, 0, 0.7);
      &::before { opacity: 0.16; }
      .process-card__tag { border-color: rgba(255, 255, 255, 0.12); color: rgba(255, 255, 255, 0.5); }
    }

    &:focus-visible { outline: 2px solid var(--color-accent); outline-offset: 4px; }

    &__inner {
      height: 100%;
      padding: clamp(2rem, 4vw, 3.5rem);
      display: flex;
      flex-direction: column;
      position: relative;
    }

    &__top { margin-bottom: auto; }

    &__badge {
      display: inline-flex;
      align-items: center;
      justify-content: center;
      width: 38px;
      height: 38px;
      border: 1px solid rgba(0, 255, 204, 0.2);
      border-radius: 10px;
      font-size: 0.68rem;
      font-weight: 800;
      color: var(--color-accent);
      letter-spacing: 0.05em;
      background: rgba(0, 255, 204, 0.04);
    }

    &__body {
      margin-bottom: 2rem;

      h3 {
        font-size: clamp(1.7rem, 3.2vw, 2.6rem);
        font-weight: 800;
        letter-spacing: -0.03em;
        line-height: 1.1;
        color: #fff;
        margin-bottom: 1.1rem;
      }

      p {
        font-size: clamp(0.9rem, 1.2vw, 1.1rem);
        color: rgba(255, 255, 255, 0.48);
        line-height: 1.75;
        max-width: 82%;
      }
    }

    &__footer {
      display: flex;
      flex-wrap: wrap;
      gap: 0.45rem;
      padding-top: 1.5rem;
      border-top: 1px solid rgba(255, 255, 255, 0.06);
      margin-top: auto;
    }

    &__tag {
      font-size: 0.6rem;
      font-weight: 600;
      letter-spacing: 0.14em;
      text-transform: uppercase;
      color: rgba(255, 255, 255, 0.28);
      padding: 0.28rem 0.7rem;
      border: 1px solid rgba(255, 255, 255, 0.07);
      border-radius: 100px;
      transition: color 0.25s ease, border-color 0.25s ease;
    }

    &__watermark {
      position: absolute;
      bottom: -0.15em;
      right: 0.1em;
      font-size: clamp(8rem, 16vw, 14rem);
      font-weight: 900;
      line-height: 1;
      letter-spacing: -0.06em;
      color: rgba(255, 255, 255, 0.028);
      pointer-events: none;
      user-select: none;
    }
  }
}

/* =========================================================================
   GALLERY SECTION
   ========================================================================= */
.gallery {
  padding: var(--section-padding);
  padding-bottom: 10vh;
  background-color: var(--color-bg);

  &__header {
    display: flex;
    justify-content: space-between;
    align-items: flex-end;
    margin-bottom: clamp(2rem, 5vh, 4rem);
    padding-bottom: 1.75rem;
    border-bottom: 1px solid rgba(255, 255, 255, 0.06);

    @media (max-width: 640px) { flex-direction: column; align-items: flex-start; gap: 1.5rem; }
  }

  &__header-left { display: flex; flex-direction: column; gap: 0.75rem; }

  &__eyebrow {
    display: block;
    font-size: 0.75rem;
    letter-spacing: 0.3em;
    text-transform: uppercase;
    color: var(--color-accent);
    font-weight: 500;
  }

  h2 {
    font-size: clamp(2.5rem, 5vw, 4.5rem);
    font-weight: 800;
    letter-spacing: -0.03em;
    line-height: 1;

    em {
      font-style: italic;
      font-weight: 300;
      background: var(--color-accent-gradient);
      -webkit-background-clip: text;
      -webkit-text-fill-color: transparent;
      background-clip: text;
    }
  }

  &__header-right {
    display: flex;
    flex-direction: column;
    align-items: flex-end;
    gap: 0.75rem;

    @media (max-width: 640px) { align-items: flex-start; }
  }

  &__count {
    font-size: 0.7rem;
    letter-spacing: 0.25em;
    text-transform: uppercase;
    color: rgba(255, 255, 255, 0.25);
  }

  &__view-all {
    display: flex;
    align-items: center;
    gap: 0.4rem;
    font-size: 0.82rem;
    letter-spacing: 0.15em;
    text-transform: uppercase;
    color: rgba(255, 255, 255, 0.4);
    text-decoration: none;
    transition: color 0.3s ease;

    .arrow { display: inline-block; transition: transform 0.3s ease; }

    &:hover { color: var(--color-accent); .arrow { transform: translate(3px, -3px); } }
  }

  &__grid {
    display: grid;
    grid-template-columns: 1.3fr 0.95fr 1.2fr;
    grid-template-rows: 38vh 38vh;
    gap: 1rem;

    @media (max-width: 992px) { grid-template-columns: 1fr 1fr; grid-template-rows: auto; }
    @media (max-width: 640px) { grid-template-columns: 1fr; }
  }

  &__item {
    background-color: #0e0630;
    clip-path: inset(100% 0% 0% 0% round 12px);
    will-change: clip-path;

    &--0 { grid-column: 1; grid-row: 1 / 3; }
    &--1 { grid-column: 2; grid-row: 1; }
    &--2 { grid-column: 2; grid-row: 2; }
    &--3 { grid-column: 3; grid-row: 1 / 3; }

    @media (max-width: 992px) {
      &--0, &--1, &--2, &--3 { grid-column: auto; grid-row: auto; aspect-ratio: 3 / 4; }
    }
  }

  &__item-inner {
    position: relative;
    width: 100%;
    height: 100%;
    overflow: hidden;
    border-radius: 12px;
    cursor: pointer;

    img {
      width: 100%;
      height: 100%;
      object-fit: cover;
      transform: scale(1.06);
      transition: transform 0.9s cubic-bezier(0.25, 1, 0.5, 1);
      display: block;
    }

    &:hover img { transform: scale(1.14); }
    &:hover .gallery__overlay-content { transform: translateY(0); opacity: 1; }
    &:hover .gallery__index { opacity: 0; }

    /* Keyboard focus */
    &:focus-visible {
      outline: 2px solid var(--color-accent);
      outline-offset: 3px;
      border-radius: 12px;
    }
  }

  &__overlay {
    position: absolute;
    inset: 0;
    background: linear-gradient(to top, rgba(0,0,0,0.88) 0%, rgba(0,0,0,0.35) 45%, transparent 72%);
    display: flex;
    align-items: flex-end;
    padding: 1.75rem;
    z-index: 2;
    pointer-events: none;
  }

  &__overlay-content {
    transform: translateY(6px);
    opacity: 0.75;
    transition: all 0.45s cubic-bezier(0.25, 1, 0.5, 1);
    width: 100%;
  }

  &__category {
    display: inline-block;
    font-size: 0.62rem;
    letter-spacing: 0.25em;
    text-transform: uppercase;
    color: var(--color-accent);
    font-weight: 600;
    margin-bottom: 0.65rem;
    padding: 0.3rem 0.75rem;
    border: 1px solid rgba(0, 255, 204, 0.3);
    border-radius: 100px;
    backdrop-filter: blur(10px);
    background: rgba(0, 255, 204, 0.06);
  }

  &__meta { display: flex; justify-content: space-between; align-items: flex-end; gap: 0.5rem; }

  &__item-title {
    font-size: clamp(0.95rem, 1.4vw, 1.35rem);
    font-weight: 600;
    color: #fff;
    letter-spacing: -0.01em;
    line-height: 1.2;
  }

  &__year {
    font-size: 0.72rem;
    color: rgba(255, 255, 255, 0.35);
    letter-spacing: 0.1em;
    font-weight: 500;
    white-space: nowrap;
    flex-shrink: 0;
  }

  &__index {
    position: absolute;
    top: 1.25rem;
    right: 1.25rem;
    font-size: 0.68rem;
    font-weight: 700;
    letter-spacing: 0.2em;
    color: rgba(255, 255, 255, 0.35);
    z-index: 3;
    transition: opacity 0.35s ease;
  }
}

/* =========================================================================
   CONTACT SECTION
   ========================================================================= */
.contact {
  padding: var(--section-padding);
  background-color: var(--color-bg);
  position: relative;

  &__container { max-width: 1400px; margin: 0 auto; }

  &__header {
    margin-bottom: clamp(3rem, 10vh, 6rem);

    h2 {
      font-size: clamp(2.5rem, 6vw, 5rem);
      font-weight: 800;
      letter-spacing: -0.03em;
    }
  }
}

/* =========================================================================
   FOOTER
   ========================================================================= */
.footer {
  padding: 4rem 5vw;
  background-color: var(--color-bg);
  border-top: 1px solid rgba(255, 255, 255, 0.05);

  &__container {
    max-width: 1400px;
    margin: 0 auto;
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 2rem;

    @media (max-width: 768px) { flex-direction: column; text-align: center; }
  }

  p {
    font-size: 0.75rem;
    letter-spacing: 0.15em;
    text-transform: uppercase;
    color: rgba(255, 255, 255, 0.3);
    font-weight: 400;
  }

  &__links {
    display: flex;
    gap: 2rem;

    a {
      font-size: 0.75rem;
      letter-spacing: 0.15em;
      text-transform: uppercase;
      text-decoration: none;
      color: rgba(255, 255, 255, 0.3);
      transition: color 0.3s ease;
      &:hover { color: var(--color-accent); }
    }
  }

  &__link--placeholder {
    opacity: 0.4;
    pointer-events: none;
    &:hover { color: rgba(255, 255, 255, 0.3) !important; }
  }
}
</style>
