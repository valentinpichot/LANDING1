<script setup lang="ts">
import { ref, computed, watch, reactive, onMounted, onUnmounted } from 'vue';
import { useI18n } from 'vue-i18n';
import ContactForm from './components/ContactForm.vue';
import gsap from 'gsap';
import ScrollTrigger from 'gsap/ScrollTrigger';
import Lenis from '@studio-freight/lenis';

gsap.registerPlugin(ScrollTrigger);

const { t, tm, locale } = useI18n();

// Persist locale + keep html[lang] in sync (#8)
watch(locale, (val) => {
  localStorage.setItem('locale', val as string);
  document.documentElement.lang = val as string;
}, { immediate: true });

const mainContainer = ref<HTMLElement | null>(null);
const heroSection   = ref<HTMLElement | null>(null);
const heroTitle     = ref<HTMLElement | null>(null);
const heroBg        = ref<HTMLElement | null>(null);
const heroBtns      = ref<HTMLElement | null>(null);
const processSection       = ref<HTMLElement | null>(null);
const processCardsWrapper  = ref<HTMLElement | null>(null);
const gallerySection  = ref<HTMLElement | null>(null);
const galleryImages   = ref<HTMLElement[]>([]);
const contactSection  = ref<HTMLElement | null>(null);
const navbar = ref<HTMLElement | null>(null);
const cursor = ref<HTMLElement | null>(null);

// Mobile menu (#2)
const menuOpen  = ref(false);
const toggleMenu = () => { menuOpen.value = !menuOpen.value; };
const closeMenu  = () => { menuOpen.value = false; };

// Preloader (#17)
const preloaderDone = ref(false);

// Hero stat counters (#16)
const heroStats = reactive({ clients: 0, projects: 0 });

// Gallery data with explicit dimensions (#4)
const galleryData = [
  { img: 'https://images.unsplash.com/photo-1618221195710-dd6b41faaea6?q=80&w=800&h=1200&auto=format&fit=crop', year: '2025', width: 800, height: 1200 },
  { img: 'https://images.unsplash.com/photo-1576091160550-2173dba999ef?q=80&w=1000&h=700&auto=format&fit=crop',  year: '2024', width: 1000, height: 700  },
  { img: 'https://images.unsplash.com/photo-1507842217343-583bb7270b66?q=80&w=1000&h=700&auto=format&fit=crop',  year: '2026', width: 1000, height: 700  },
  { img: 'https://images.unsplash.com/photo-1536440136628-849c177e76a1?q=80&w=800&h=1200&auto=format&fit=crop',  year: '2025', width: 800, height: 1200 },
];

const processSteps = computed(
  () => tm('process.steps') as Array<{ title: string; description: string }>
);

// processTags now come from i18n (#13)
const processTags = computed(
  () => tm('process.tags') as Array<string[]>
);

let ctx: gsap.Context;
let lenis: Lenis | null = null;
let cursorMoveHandler: ((e: MouseEvent) => void) | null = null;
let lenisTickerFn:    ((time: number) => void) | null = null;  // fix #1
let escHandler:       ((e: KeyboardEvent) => void) | null = null;

const scrollTo = (target: string) => {
  lenis?.scrollTo(target, { duration: 1 });
  closeMenu();
};

// Scroll to the right page position so GSAP reveals a pinned process card (#10)
const scrollToProcessCard = (index: number) => {
  if (!processSection.value || !processCardsWrapper.value || !lenis) return;
  const totalWidth = processCardsWrapper.value.scrollWidth - window.innerWidth;
  const fraction   = processSteps.value.length > 1 ? index / (processSteps.value.length - 1) : 0;
  lenis.scrollTo(processSection.value.offsetTop + fraction * totalWidth, { duration: 0.8 });
};

onMounted(() => {
  // Preloader: hide after 950 ms — let the ring animation complete (#17)
  setTimeout(() => { preloaderDone.value = true; }, 950);

  // Close mobile menu on Escape (#2)
  escHandler = (e: KeyboardEvent) => { if (e.key === 'Escape') closeMenu(); };
  window.addEventListener('keydown', escHandler);

  // Initialize Lenis
  lenis = new Lenis({
    duration: 1.2,
    easing: (t) => Math.min(1, 1.001 - Math.pow(2, -10 * t)),
    orientation: 'vertical',
    gestureOrientation: 'vertical',
    smoothWheel: true,
    wheelMultiplier: 1,
    touchMultiplier: 2,
    infinite: false,
  });

  lenis.on('scroll', ScrollTrigger.update);

  // Store reference so we can properly remove it (#1)
  lenisTickerFn = (time) => { lenis?.raf(time * 1000); };
  gsap.ticker.add(lenisTickerFn);
  gsap.ticker.lagSmoothing(0);

  if (mainContainer.value) {
    ctx = gsap.context(() => {
      // 1. Hero Effects
      const tl = gsap.timeline();

      if (heroBg.value && heroSection.value) {
        gsap.to(heroBg.value, {
          scale: 1.15,
          duration: 20,
          ease: 'none',
          repeat: -1,
          yoyo: true,
        });

        gsap.to(heroBg.value, {
          yPercent: 20,
          ease: 'none',
          scrollTrigger: {
            trigger: heroSection.value,
            start: 'top top',
            end: 'bottom top',
            scrub: true,
          },
        });
      }

      if (heroTitle.value) {
        const letters = heroTitle.value.querySelectorAll('.char');

        tl.from(letters, {
          y: '100%',
          opacity: 0,
          duration: 1.2,
          stagger: 0.04,
          ease: 'power4.out',
          delay: 0.2,
        })
          .from('.hero__subtitle', {
            y: 20,
            autoAlpha: 0,
            duration: 0.8,
            ease: 'power3.out',
          }, '-=0.6');

        if (heroBtns.value) {
          const btns = heroBtns.value.querySelectorAll('.btn');
          tl.fromTo(btns,
            { y: 20, autoAlpha: 0 },
            {
              y: 0,
              autoAlpha: 1,
              duration: 0.8,
              stagger: 0.15,
              ease: 'power3.out',
              clearProps: 'all',
            },
            '-=0.4'
          );
        }

        // Infobar reveal then immediately kick off counter (#16)
        tl.from('.hero__infobar', {
          autoAlpha: 0,
          y: 20,
          duration: 1,
          ease: 'power3.out',
        }, '-=0.4')
          .to(heroStats, {
            clients: 80,
            projects: 150,
            duration: 2,
            ease: 'power2.out',
            snap: { clients: 1, projects: 1 },
          }, '<0.3');
      }

      if (navbar.value) {
        gsap.fromTo(navbar.value,
          { y: -20, autoAlpha: 0 },
          {
            y: 0,
            autoAlpha: 1,
            duration: 1,
            ease: 'power3.out',
            delay: 0.8,
            clearProps: 'all',
          }
        );

        ScrollTrigger.create({
          start: 'top -50',
          onEnter:     () => navbar.value?.classList.add('navbar--scrolled'),
          onLeaveBack: () => navbar.value?.classList.remove('navbar--scrolled'),
        });
      }

      // 2. Process Section: Horizontal Scroll over Vertical Scroll
      if (processCardsWrapper.value && processSection.value) {
        const totalWidth = processCardsWrapper.value.scrollWidth - window.innerWidth;

        gsap.to(processCardsWrapper.value, {
          x: -totalWidth,
          ease: 'none',
          scrollTrigger: {
            trigger: processSection.value,
            pin: true,
            scrub: 1,
            start: 'top top',
            end: () => `+=${totalWidth}`,
          },
        });
      }

      // 3. Gallery: Curtain wipe reveal
      if (galleryImages.value.length > 0) {
        ScrollTrigger.batch(galleryImages.value, {
          onEnter: (elements) => {
            gsap.fromTo(elements,
              { clipPath: 'inset(100% 0% 0% 0% round 12px)' },
              {
                clipPath: 'inset(0% 0% 0% 0% round 12px)',
                duration: 1.4,
                stagger: 0.18,
                ease: 'power3.out',
                overwrite: true,
              }
            );
          },
          start: 'top 85%',
        });
      }

      // 4. Global Text Reveal on Scroll
      const textElements = mainContainer.value?.querySelectorAll('.reveal-text');
      if (textElements) {
        textElements.forEach((el) => {
          gsap.fromTo(el,
            { autoAlpha: 0, y: -20 },
            {
              autoAlpha: 1,
              y: 0,
              duration: 1,
              ease: 'power3.out',
              scrollTrigger: {
                trigger: el,
                start: 'top 60%',
                toggleActions: 'play none none reverse',
              },
            }
          );
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
  if (ctx) ctx.revert();
  if (lenis) { lenis.destroy(); lenis = null; }
  if (cursorMoveHandler) window.removeEventListener('mousemove', cursorMoveHandler);
  // Use stored reference — not a new anonymous function (#1)
  if (lenisTickerFn)  { gsap.ticker.remove(lenisTickerFn); lenisTickerFn = null; }
  if (escHandler)     { window.removeEventListener('keydown', escHandler); escHandler = null; }
});
</script>

<template>
  <!-- Preloader (#17) -->
  <Transition name="preloader-fade">
    <div v-if="!preloaderDone" class="preloader" aria-hidden="true">
      <div class="preloader__ring-wrap">
        <svg class="preloader__ring" viewBox="0 0 100 100" aria-hidden="true">
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

  <!-- Mobile menu overlay (#2) -->
  <Transition name="mobile-menu">
    <div v-if="menuOpen" class="mobile-menu" role="dialog" aria-modal="true" :aria-label="t('nav.open_menu')">
      <div class="mobile-menu__backdrop" @click="closeMenu" aria-hidden="true"></div>
      <nav class="mobile-menu__content">
        <button class="mobile-menu__close" @click="closeMenu" :aria-label="t('nav.close_menu')">✕</button>
        <a href="#process" class="mobile-menu__link" @click.prevent="scrollTo('#process')">{{ t('nav.approach') }}</a>
        <a href="#gallery"  class="mobile-menu__link" @click.prevent="scrollTo('#gallery')">{{ t('nav.gallery') }}</a>
        <a href="#contact"  class="mobile-menu__link" @click.prevent="scrollTo('#contact')">{{ t('nav.project') }}</a>
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
          <a href="#process" class="nav-link" @click.prevent="scrollTo('#process')">{{ t('nav.approach') }}</a>
          <a href="#gallery"  class="nav-link" @click.prevent="scrollTo('#gallery')">{{ t('nav.gallery') }}</a>
          <a href="#contact"  class="nav-link" @click.prevent="scrollTo('#contact')">{{ t('nav.project') }}</a>
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
          <!-- Hamburger (mobile only) (#2) -->
          <button
            class="navbar__hamburger"
            @click="toggleMenu"
            :aria-label="t('nav.open_menu')"
            :aria-expanded="menuOpen"
          >
            <span></span>
            <span></span>
          </button>
        </div>
      </div>
    </nav>

    <!-- HERO SECTION -->
    <section ref="heroSection" class="hero" id="home">
      <div ref="heroBg" class="hero__bg"></div>
      <div class="hero__overlay"></div>

      <div class="hero__content">
        <h1 ref="heroTitle" class="hero__title">
          <span class="word">
            <span class="char">V</span><span class="char">-</span><span class="char">S</span><span class="char">T</span><span class="char">U</span><span class="char">D</span><span class="char">I</span><span class="char">O</span><span class="char">S</span>
          </span>
          <span class="word">
            <span class="char">D</span><span class="char">E</span><span class="char">M</span><span class="char">O</span>
          </span>
        </h1>
        <p class="hero__subtitle">{{ t('hero.subtitle') }}</p>

        <div ref="heroBtns" class="hero__ctas">
          <button class="btn btn--primary" @click="scrollTo('#contact')">
            <span>{{ t('hero.cta_primary') }}</span>
            <div class="btn-glow"></div>
          </button>
          <button class="btn btn--secondary" @click="scrollTo('#gallery')">{{ t('hero.cta_secondary') }}</button>
        </div>
      </div>

      <!-- SCROLL INDICATOR -->
      <div class="scroll-indicator" aria-hidden="true">
        <div class="scroll-indicator__line">
          <div class="scroll-indicator__drop"></div>
        </div>
      </div>

      <!-- HERO INFO BAR — animated counters (#16) -->
      <div class="hero__infobar">
        <div class="info-item">
          <span class="label">{{ t('hero.stat_clients_label') }}</span>
          <span class="value">{{ heroStats.clients }}<span class="stat-plus">+</span></span>
        </div>
        <div class="info-item">
          <span class="label">{{ t('hero.stat_projects_label') }}</span>
          <span class="value">{{ heroStats.projects }}<span class="stat-plus">+</span></span>
        </div>
        <div class="info-item">
          <span class="label">{{ t('hero.stat_since_label') }}</span>
          <span class="value">{{ t('hero.stat_since_value') }}</span>
        </div>
      </div>
    </section>

    <!-- PROCESS SECTION -->
    <section ref="processSection" class="process" id="process">
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

    <!-- GALLERY SECTION -->
    <section ref="gallerySection" class="gallery" id="gallery">
      <div class="gallery__header">
        <div class="gallery__header-left">
          <span class="gallery__eyebrow">{{ t('gallery.eyebrow') }}</span>
          <h2 class="reveal-text">{{ t('gallery.title_main') }} <em>{{ t('gallery.title_accent') }}</em></h2>
        </div>
        <div class="gallery__header-right">
          <span class="gallery__count reveal-text">{{ t('gallery.count', { count: galleryData.length }) }}</span>
          <!-- "View all" → scroll to contact instead of # (#15) -->
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
          <div class="gallery__item-inner">
            <!-- First image eager, rest lazy (#3); explicit dimensions for CLS (#4) -->
            <img
              :src="item.img"
              :alt="t(`gallery.projects.${index}.title`)"
              :loading="index === 0 ? 'eager' : 'lazy'"
              :width="item.width"
              :height="item.height"
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

    <!-- CONTACT SECTION -->
    <section ref="contactSection" class="contact" id="contact">
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
            <!-- Placeholder links disabled until real pages exist (#15) -->
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

/* Global focus ring (#6) */
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

  &.has-smooth-scrolling {
    overflow: hidden;
  }
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
   PRELOADER (#17)
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
  gap: 1.5rem;
  pointer-events: none;
}

.preloader__dot {
  width: 10px;
  height: 10px;
  border-radius: 50%;
  background: var(--color-accent);
  box-shadow: 0 0 20px var(--color-accent), 0 0 40px rgba(0, 255, 204, 0.3);
  animation: pulse-dot 1.2s ease-in-out infinite;
}

.preloader__text {
  font-size: 0.7rem;
  font-weight: 800;
  letter-spacing: 0.45em;
  color: rgba(255, 255, 255, 0.3);
  text-transform: uppercase;
}

.preloader-fade-leave-active { transition: opacity 0.5s ease; }
.preloader-fade-leave-to     { opacity: 0; }

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
  box-shadow:
    inset 0 1px 0 rgba(255,255,255,0.06),
    0 8px 32px rgba(0,0,0,0.5),
    0 0 0 0 rgba(0,255,204,0);
  transition: all 0.5s cubic-bezier(0.25, 1, 0.5, 1);
  will-change: transform, width, box-shadow;

  &--scrolled {
    padding: 0.5rem 0.5rem 0.5rem 1.25rem;
    background: rgba(4, 4, 4, 0.85);
    width: 84%;
    border-color: rgba(0, 255, 204, 0.14);
    box-shadow:
      inset 0 1px 0 rgba(0,255,204,0.06),
      0 16px 48px rgba(0,0,0,0.7),
      0 0 60px rgba(0,255,204,0.03);
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
      /* Raised from 0.38 to 0.55 for WCAG AA compliance (#9) */
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

  /* Hamburger button (#2) */
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
      transition: background 0.2s ease;
    }

    &:hover {
      background: rgba(255, 255, 255, 0.06);
      span { background: #fff; }
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

    &:hover:not(.lang-btn--active) {
      color: rgba(255, 255, 255, 0.6);
      background: rgba(255, 255, 255, 0.05);
    }

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
   MOBILE MENU (#2)
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

    &:hover { color: var(--color-accent); }
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

.mobile-menu-enter-active .mobile-menu__content {
  transition: transform 0.35s cubic-bezier(0.25, 1, 0.5, 1);
}
.mobile-menu-enter-from .mobile-menu__content { transform: translateX(100%); }

.mobile-menu-leave-active .mobile-menu__content {
  transition: transform 0.25s ease;
}
.mobile-menu-leave-to .mobile-menu__content { transform: translateX(100%); }

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
.landing-page * {
  cursor: none;
}

.landing-page input,
.landing-page textarea {
  cursor: text;
}

/* Restore visible cursor on keyboard focus (#6) */
.landing-page *:focus-visible {
  cursor: auto;
}

.landing-page { width: 100%; }

.reveal-text {
  visibility: hidden;
  opacity: 0;
  will-change: transform, opacity;
}

/* =========================================================================
   HERO SECTION
   ========================================================================= */
.hero {
  position: relative;
  height: 100vh;
  width: 100vw;
  overflow: hidden;
  @include flex-center;

  &__bg {
    position: absolute;
    top: -10%;
    left: -10%;
    width: 120%;
    height: 120%;
    background:
      radial-gradient(ellipse at 20% 20%, rgba(130, 80, 255, 1)     0%, transparent 42%),
      radial-gradient(ellipse at 75% 10%, rgba(60,  150, 255, 1)     0%, transparent 40%),
      radial-gradient(ellipse at 50% 55%, rgba(255, 255, 255, 0.06)  0%, transparent 50%),
      radial-gradient(ellipse at 58% 82%, rgba(0,   210, 255, 0.90)  0%, transparent 44%),
      radial-gradient(ellipse at 10% 78%, rgba(200, 40,  255, 0.85)  0%, transparent 40%),
      radial-gradient(ellipse at 88% 60%, rgba(90,  30,  230, 0.75)  0%, transparent 36%),
      #10082e;
    z-index: 0;
    will-change: transform;
  }

  &::after {
    content: '';
    position: absolute;
    inset: 0;
    z-index: 3;
    pointer-events: none;
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='240' height='240'%3E%3Cfilter id='n'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.8' numOctaves='4' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='240' height='240' filter='url(%23n)'/%3E%3C/svg%3E");
    opacity: 0.055;
  }

  &__overlay {
    position: absolute;
    inset: 0;
    background: linear-gradient(180deg, rgba(16, 8, 46, 0.0) 0%, rgba(16, 8, 46, 0.22) 50%, rgba(16, 8, 46, 0.97) 100%);
    z-index: 1;
  }

  &__content {
    text-align: center;
    position: relative;
    z-index: 10;
    padding: 0 5vw;
  }

  &__title {
    font-size: clamp(3rem, 10vw, 9rem);
    font-weight: 800;
    text-transform: uppercase;
    letter-spacing: -0.04em;
    line-height: 1;
    display: flex;
    flex-wrap: wrap;
    justify-content: center;
    gap: 0.25em;

    .word {
      display: inline-flex;
      overflow: hidden;
    }

    .char {
      display: inline-block;
      will-change: transform, opacity;
      transform: translateY(100%);
      background: linear-gradient(to bottom, #ffffff 40%, rgba(255, 255, 255, 0.5) 100%);
      -webkit-background-clip: text;
      -webkit-text-fill-color: transparent;
      background-clip: text;
    }
  }

  &__subtitle {
    margin-top: 1.5rem;
    font-size: clamp(1rem, 2vw, 1.5rem);
    font-weight: 300;
    letter-spacing: 0.2em;
    color: rgba(255, 255, 255, 0.6);
    text-transform: uppercase;
    margin-bottom: 3rem;
  }

  &__ctas {
    display: flex;
    justify-content: center;
    gap: 1.5rem;

    @media (max-width: 640px) {
      flex-direction: column;
      align-items: center;
      gap: 1rem;
    }

    .btn {
      padding: 1.2rem 3rem;
      border-radius: 100px;
      font-size: 1rem;
      font-weight: 600;
      cursor: pointer;
      text-transform: uppercase;
      letter-spacing: 0.1em;
      transition: all 0.3s cubic-bezier(0.25, 1, 0.5, 1);
      position: relative;
      overflow: hidden;

      &--primary {
        background: #fff;
        color: #000;
        border: none;

        span { position: relative; z-index: 2; }

        .btn-glow {
          position: absolute;
          inset: 0;
          background: var(--color-accent-gradient);
          opacity: 0;
          transition: opacity 0.3s ease;
          z-index: 1;
        }

        &:hover {
          transform: translateY(-5px);
          .btn-glow { opacity: 1; }
        }
      }

      &--secondary {
        background: transparent;
        color: #fff;
        border: 1px solid rgba(255, 255, 255, 0.2);

        &:hover {
          background: rgba(255, 255, 255, 0.05);
          border-color: #fff;
          transform: translateY(-5px);
        }
      }
    }
  }

  .scroll-indicator {
    position: absolute;
    bottom: 7rem;
    left: 50%;
    transform: translateX(-50%);
    display: flex;
    flex-direction: column;
    align-items: center;
    z-index: 15;

    @media (max-width: 768px) { display: none; }

    &__line {
      width: 1px;
      height: 64px;
      background: rgba(255, 255, 255, 0.08);
      position: relative;
      overflow: hidden;
      border-radius: 1px;
    }

    &__drop {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 40%;
      background: linear-gradient(to bottom, var(--color-accent), transparent);
      animation: scroll-drop 2.2s cubic-bezier(0.4, 0, 0.6, 1) infinite;
    }
  }

  @keyframes scroll-drop {
    0%   { top: -40%; opacity: 1; }
    80%  { opacity: 0.6; }
    100% { top: 120%; opacity: 0; }
  }

  &__infobar {
    position: absolute;
    bottom: 0;
    left: 0;
    width: 100%;
    padding: 2.5rem 5vw;
    display: flex;
    justify-content: center;
    gap: clamp(2rem, 10vw, 8rem);
    background: linear-gradient(to top, rgba(5, 5, 5, 0.8), transparent);
    border-top: 1px solid rgba(255, 255, 255, 0.05);
    backdrop-filter: blur(10px);
    z-index: 20;

    @media (max-width: 768px) {
      flex-wrap: wrap;
      gap: 2rem;
      padding: 1.5rem 5vw;
    }

    .info-item {
      display: flex;
      flex-direction: column;
      gap: 0.4rem;
      text-align: left;

      .label {
        font-size: 0.7rem;
        text-transform: uppercase;
        letter-spacing: 0.2em;
        color: rgba(255, 255, 255, 0.4);
      }

      .value {
        font-size: 1.1rem;
        font-weight: 700;
        color: #fff;
        letter-spacing: 0.05em;
        font-variant-numeric: tabular-nums;
      }

      .stat-plus {
        font-size: 0.85em;
        opacity: 0.7;
      }
    }
  }
}

/* =========================================================================
   PROCESS SECTION (PINNED HORIZONTAL SCROLL)
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
    transition:
      transform   0.45s cubic-bezier(0.25, 1, 0.5, 1),
      box-shadow  0.45s cubic-bezier(0.25, 1, 0.5, 1);

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

    /* Keyboard focus highlight (#10) */
    &:focus-visible {
      outline: 2px solid var(--color-accent);
      outline-offset: 4px;
    }

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

    &:hover {
      color: var(--color-accent);
      .arrow { transform: translate(3px, -3px); }
    }
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
  }

  &__overlay {
    position: absolute;
    inset: 0;
    background: linear-gradient(to top, rgba(0,0,0,0.88) 0%, rgba(0,0,0,0.35) 45%, transparent 72%);
    display: flex;
    align-items: flex-end;
    padding: 1.75rem;
    z-index: 2;
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

  /* Placeholder links (#15) */
  &__link--placeholder {
    opacity: 0.4;
    pointer-events: none;

    &:hover { color: rgba(255, 255, 255, 0.3) !important; }
  }
}
</style>
