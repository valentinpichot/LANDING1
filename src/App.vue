<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';
import gsap from 'gsap';
import ScrollTrigger from 'gsap/ScrollTrigger';
import Lenis from '@studio-freight/lenis';

gsap.registerPlugin(ScrollTrigger);

const mainContainer = ref<HTMLElement | null>(null);
const heroSection = ref<HTMLElement | null>(null);
const heroTitle = ref<HTMLElement | null>(null);
const heroBg = ref<HTMLElement | null>(null);
const heroBtns = ref<HTMLElement | null>(null);

const processSection = ref<HTMLElement | null>(null);
const processCardsWrapper = ref<HTMLElement | null>(null);

const gallerySection = ref<HTMLElement | null>(null);
const galleryImages = ref<HTMLElement[]>([]);
const contactSection = ref<HTMLElement | null>(null);

const galleryData = [
  { id: 1015, title: 'Identité Visuelle', category: 'Branding', year: '2026', w: 800, h: 1200 },
  { id: 1016, title: 'Interface Digitale', category: 'UI/UX', year: '2025', w: 1000, h: 700 },
  { id: 1025, title: 'Direction Artistique', category: 'Editorial', year: '2026', w: 1000, h: 700 },
  { id: 1032, title: 'Motion Design', category: 'Animation', year: '2025', w: 800, h: 1200 },
];
const navbar = ref<HTMLElement | null>(null);

let ctx: gsap.Context;
let lenis: Lenis | null = null;

onMounted(() => {
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

  // Synchronize Lenis with GSAP ScrollTrigger
  lenis.on('scroll', ScrollTrigger.update);

  gsap.ticker.add((time) => {
    lenis?.raf(time * 1000);
  });

  gsap.ticker.lagSmoothing(0);
  
  if (mainContainer.value) {
    ctx = gsap.context(() => {
    // 1. Hero Effects
    const tl = gsap.timeline();

    // Background continuous zoom + Parallax
    if (heroBg.value && heroSection.value) {
      // Continuous subtle zoom
      gsap.to(heroBg.value, {
        scale: 1.15,
        duration: 20, // Slower for more elegance
        ease: "none",
        repeat: -1,
        yoyo: true
      });

      // Scroll Parallax
      gsap.to(heroBg.value, {
        yPercent: 20,
        ease: "none",
        scrollTrigger: {
          trigger: heroSection.value,
          start: "top top",
          end: "bottom top",
          scrub: true
        }
      });
    }

    // Split-like text reveal + CTAs sequence
    if (heroTitle.value) {
      const letters = heroTitle.value.querySelectorAll('.char');

      tl.from(letters, {
        y: '100%',
        opacity: 0,
        duration: 1.2,
        stagger: 0.04,
        ease: 'power4.out',
        delay: 0.2
      })
        .from('.hero__subtitle', {
          y: 20,
          autoAlpha: 0,
          duration: 0.8,
          ease: 'power3.out'
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
            clearProps: "all"
          },
          '-=0.4'
        );
      }

      tl.from('.hero__infobar', {
        autoAlpha: 0,
        y: 20,
        duration: 1,
        ease: 'power3.out'
      }, '-=0.4');
    }

    // Navbar entrance (Independent of heroTitle presence)
    if (navbar.value) {
      gsap.fromTo(navbar.value,
        { y: -20, autoAlpha: 0 },
        {
          y: 0,
          autoAlpha: 1,
          duration: 1,
          ease: 'power3.out',
          delay: 0.8, // Wait for hero initiation
          clearProps: "all"
        }
      );

      // Scrolled state effect
      ScrollTrigger.create({
        start: "top -50",
        onEnter: () => navbar.value?.classList.add('navbar--scrolled'),
        onLeaveBack: () => navbar.value?.classList.remove('navbar--scrolled'),
      });
    }

    // 2. Process Section: Horizontal Scroll over Vertical Scroll
    if (processCardsWrapper.value && processSection.value) {
      const totalWidth = processCardsWrapper.value.scrollWidth - window.innerWidth;

      gsap.to(processCardsWrapper.value, {
        x: -totalWidth,
        ease: "none",
        scrollTrigger: {
          trigger: processSection.value,
          pin: true,
          scrub: 1,
          start: "top top",
          end: () => `+=${totalWidth}`
        }
      });
    }

    // 3. Reveal Gallery: Curtain wipe reveal
    if (galleryImages.value.length > 0) {
      ScrollTrigger.batch(galleryImages.value, {
        onEnter: (elements) => {
          gsap.fromTo(elements,
            { clipPath: "inset(100% 0% 0% 0% round 12px)" },
            {
              clipPath: "inset(0% 0% 0% 0% round 12px)",
              duration: 1.4,
              stagger: 0.18,
              ease: "power3.out",
              overwrite: true
            }
          );
        },
        start: "top 85%",
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
            ease: "power3.out",
            scrollTrigger: {
              trigger: el,
              start: "top 60%",
              toggleActions: "play none none reverse"
            }
          }
        );
      });
    }

    }, mainContainer.value); // Scope to main component
  }
});

onUnmounted(() => {
  if (ctx) ctx.revert();
  if (lenis) {
    lenis.destroy();
    lenis = null;
  }
  gsap.ticker.remove((time) => {
    lenis?.raf(time * 1000);
  });
});
</script>

<template>
  <main ref="mainContainer" class="landing-page dark-mode">
    <!-- NAVIGATION -->
    <nav ref="navbar" class="navbar">
      <div class="navbar__container">
        <a href="#" class="navbar__logo">VISION PURE</a>

        <div class="navbar__menu">
          <a href="#process" class="nav-link" @click.prevent="lenis?.scrollTo('#process', { duration: 1 })">Approche</a>
          <a href="#gallery" class="nav-link" @click.prevent="lenis?.scrollTo('#gallery', { duration: 1 })">Galerie</a>
          <a href="#contact" class="nav-link" @click.prevent="lenis?.scrollTo('#contact', { duration: 1 })">Projet</a>
        </div>

        <button class="navbar__cta" @click="lenis?.scrollTo('#contact', { duration: 1 })">
          <span>Contact</span>
        </button>
      </div>
    </nav>

    <!-- HERO SECTION -->
    <section ref="heroSection" class="hero" id="home">
      <div ref="heroBg" class="hero__bg"></div>
      <div class="hero__overlay"></div>

      <div class="hero__content">
        <h1 ref="heroTitle" class="hero__title">
          <span class="word">
            <span class="char">V</span><span class="char">I</span><span class="char">S</span><span
              class="char">I</span><span class="char">O</span><span class="char">N</span>
          </span>
          <span class="word">
            <span class="char">P</span><span class="char">U</span><span class="char">R</span><span class="char">E</span>
          </span>
        </h1>
        <p class="hero__subtitle">L'excellence visuelle, redéfinie.</p>

        <div ref="heroBtns" class="hero__ctas">
          <button class="btn btn--primary">
            <span>Lancer un projet</span>
            <div class="btn-glow"></div>
          </button>
          <button class="btn btn--secondary">Notre expertise</button>
        </div>
      </div>

      <!-- HERO INFO BAR -->
      <div class="hero__infobar">
        <div class="info-item">
          <span class="label">Performance</span>
          <span class="value">60FPS</span>
        </div>
        <div class="info-item">
          <span class="label">Expérience</span>
          <span class="value">IMMERSIVE</span>
        </div>
        <div class="info-item">
          <span class="label">Animation</span>
          <span class="value">GSAP 3</span>
        </div>
      </div>
    </section>

    <!-- PROCESS SECTION -->
    <section ref="processSection" class="process" id="process">
      <div class="process__header">
        <h2 class="reveal-text">L'Approche</h2>
      </div>
      <div ref="processCardsWrapper" class="process__cards">
        <div class="process-card" v-for="i in 3" :key="i">
          <div class="process-card__inner">
            <span class="process-card__number">0{{ i }}</span>
            <h3 class="reveal-text">{{ ['Conception', 'Expérience', 'Performance'][i - 1] }}</h3>
            <p class="reveal-text">Découvrez notre maîtrise de l'esthétique et de la technologie pour sublimer chaque
              détail de cette création digitale fluide et performante.</p>
          </div>
        </div>
      </div>
    </section>

    <!-- GALLERY SECTION -->
    <section ref="gallerySection" class="gallery" id="gallery">
      <div class="gallery__header">
        <div class="gallery__header-left">
          <span class="gallery__eyebrow">— Réalisations</span>
          <h2 class="reveal-text">Galerie <em>Immersive</em></h2>
        </div>
        <div class="gallery__header-right">
          <span class="gallery__count reveal-text">{{ galleryData.length }} projets</span>
          <a href="#" class="gallery__view-all reveal-text">Voir tout <span class="arrow">↗</span></a>
        </div>
      </div>

      <div class="gallery__grid">
        <div
          v-for="(item, index) in galleryData"
          :key="item.id"
          class="gallery__item"
          :class="`gallery__item--${index}`"
          ref="galleryImages"
        >
          <div class="gallery__item-inner">
            <img
              :src="`https://picsum.photos/id/${item.id}/${item.w}/${item.h}`"
              :alt="item.title"
              loading="lazy"
            />
            <div class="gallery__overlay">
              <div class="gallery__overlay-content">
                <span class="gallery__category">{{ item.category }}</span>
                <div class="gallery__meta">
                  <h3 class="gallery__item-title">{{ item.title }}</h3>
                  <span class="gallery__year">{{ item.year }}</span>
                </div>
              </div>
            </div>
            <span class="gallery__index">0{{ index + 1 }}</span>
          </div>
        </div>
      </div>
    </section>

    <!-- CONTACT SECTION -->
    <section ref="contactSection" class="contact" id="contact">
      <div class="contact__container">
        <div class="contact__header">
          <h2 class="reveal-text">Parlons de votre projet</h2>
        </div>

        <div class="contact__layout">
          <!-- Info Column -->
          <div class="contact__info">
            <div class="info-block">
              <span class="info-label reveal-text">Email</span>
              <a href="mailto:hello@visionpure.studio" class="info-value reveal-text">hello@visionpure.studio</a>
            </div>
            <div class="info-block">
              <span class="info-label reveal-text">Location</span>
              <p class="info-value reveal-text">Paris, France<br />Worldwide</p>
            </div>
            <div class="info-block">
              <span class="info-label reveal-text">Social</span>
              <div class="social-links reveal-text">
                <a href="#">Instagram</a>
                <a href="#">LinkedIn</a>
                <a href="#">Dribbble</a>
              </div>
            </div>
          </div>

          <!-- Form Column -->
          <form class="contact__form reveal-text" @submit.prevent>
            <div class="form-group">
              <input type="text" id="name" placeholder=" " required />
              <label for="name">Nom</label>
            </div>
            <div class="form-group">
              <input type="email" id="email" placeholder=" " required />
              <label for="email">Email</label>
            </div>
            <div class="form-group">
              <textarea id="message" rows="4" placeholder=" " required></textarea>
              <label for="message">Message</label>
            </div>
            <button type="submit" class="submit-btn">
              <span>Envoyer</span>
              <div class="btn-glow"></div>
            </button>
          </form>
        </div>
      </div>
    </section>

    <!-- FOOTER -->
    <footer class="footer">
      <div class="footer__container">
        <div class="footer__left">
          <p class="">© 2026 VISION PURE STUDIO</p>
        </div>
        <div class="footer__center">
          <p class="">DESIGNED WITH PASSION</p>
        </div>
        <div class="footer__right">
          <div class="footer__links ">
            <a href="#">Privacy</a>
            <a href="#">Terms</a>
          </div>
        </div>
      </div>
    </footer>
  </main>
</template>

<style lang="scss">
:root {
  /* Modern CSS Variables & Dark Mode / Neon theme */
  --color-bg: #050505;
  --color-text: #f0f0f0;
  --color-accent: #00ffcc;
  /* Neon cyan accent */
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

body {
  background-color: var(--color-bg);
  color: var(--color-text);
  font-family: var(--font-primary);
  overflow-x: hidden;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;

  /* Smooth Scroll specific fixes */
  &.has-smooth-scrolling {
    overflow: hidden;
  }
}

html.lenis {
  height: auto;
}

.lenis.lenis-smooth {
  scroll-behavior: auto;
}

.lenis.lenis-smooth [data-lenis-prevent] {
  overscroll-behavior: contain;
}

.lenis.lenis-stopped {
  overflow: hidden;
}

.lenis.lenis-scrolling iframe {
  pointer-events: none;
}

/* Base Mixins */
@mixin flex-center {
  display: flex;
  justify-content: center;
  align-items: center;
}

/* =========================================================================
   NAVIGATION (DETACHED GLASSMORPHISM)
   ========================================================================= */
.navbar {
  position: fixed;
  top: 1.5rem;
  left: 50%;
  transform: translateX(-50%);
  width: 90%;
  max-width: 1200px;
  z-index: 100;
  padding: 0.75rem 1.5rem;
  background: rgba(255, 255, 255, 0.03);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  border: 1px solid rgba(255, 255, 255, 0.05);
  border-radius: 100px;
  transition: all 0.4s cubic-bezier(0.25, 1, 0.5, 1);
  will-change: transform, background, padding;

  &--scrolled {
    padding: 0.5rem 1.2rem;
    background: rgba(0, 0, 0, 0.4);
    width: 85%;
    border-color: rgba(0, 255, 204, 0.2);
  }

  &__container {
    display: flex;
    justify-content: space-between;
    align-items: center;
  }

  &__logo {
    font-size: 1.1rem;
    font-weight: 800;
    letter-spacing: 0.1em;
    color: #fff;
    text-decoration: none;
    transition: color 0.3s ease;

    &:hover {
      color: var(--color-accent);
    }
  }

  &__menu {
    display: flex;
    gap: 2.5rem;

    @media (max-width: 768px) {
      display: none;
    }

    .nav-link {
      font-size: 0.75rem;
      font-weight: 500;
      text-transform: uppercase;
      letter-spacing: 0.2em;
      color: rgba(255, 255, 255, 0.5);
      text-decoration: none;
      transition: all 0.3s ease;
      position: relative;

      &::after {
        content: '';
        position: absolute;
        bottom: -4px;
        left: 0;
        width: 0;
        height: 1px;
        background: var(--color-accent);
        transition: width 0.3s ease;
      }

      &:hover {
        color: #fff;

        &::after {
          width: 100%;
        }
      }
    }
  }

  &__cta {
    background: var(--color-accent);
    color: #000;
    border: none;
    padding: 0.5rem 1.5rem;
    border-radius: 100px;
    font-size: 0.75rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    cursor: pointer;
    transition: transform 0.3s ease, box-shadow 0.3s ease;

    &:hover {
      transform: scale(1.05);
      box-shadow: 0 0 20px rgba(0, 255, 204, 0.4);
    }
  }
}

.landing-page {
  width: 100%;
}

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
    /* Cinematic Background */
    background-image: url('https://images.unsplash.com/photo-1550684848-fac1c5b4e853?q=80&w=2070&auto=format&fit=crop');
    background-size: cover;
    background-position: center;
    z-index: 0;
    will-change: transform;
  }

  &__overlay {
    position: absolute;
    inset: 0;
    background: linear-gradient(180deg, rgba(5, 5, 5, 0.4) 0%, rgba(5, 5, 5, 0.95) 100%);
    z-index: 1;
  }

  &__content {
    text-align: center;
    position: relative;
    /* Necessary for z-index to work */
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
      /* Crucial for the Split-type upward mask reveal */
    }

    .char {
      display: inline-block;
      will-change: transform, opacity;
      transform: translateY(100%);
      /* Subtle gradient text fill for premium feel */
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

        span {
          position: relative;
          z-index: 2;
        }

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

          .btn-glow {
            opacity: 1;
          }
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

    h2 {
      font-size: clamp(2.5rem, 5vw, 4rem);
      font-weight: 700;
      letter-spacing: -0.02em;
      background: var(--color-accent-gradient);
      -webkit-background-clip: text;
      -webkit-text-fill-color: transparent;
      display: inline-block;
    }
  }

  &__cards {
    display: flex;
    width: max-content;
    padding: 0 5vw;
    gap: clamp(2rem, 5vw, 4rem);
    will-change: transform;
  }

  &-card {
    width: clamp(300px, 60vw, 700px);
    height: 60vh;
    border-radius: 32px;
    background: linear-gradient(135deg, rgba(255, 255, 255, 0.03) 0%, rgba(255, 255, 255, 0.01) 100%);
    backdrop-filter: blur(20px);
    border: 1px solid rgba(255, 255, 255, 0.05);
    padding: clamp(2rem, 5vw, 4rem);
    display: flex;
    flex-direction: column;
    justify-content: center;
    box-shadow: 0 30px 60px rgba(0, 0, 0, 0.8);
    position: relative;
    overflow: hidden;

    &::after {
      content: '';
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      height: 1px;
      background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.2), transparent);
    }

    &__number {
      font-size: clamp(4rem, 8vw, 6rem);
      font-weight: 800;
      color: var(--color-accent);
      opacity: 0.15;
      line-height: 1;
      margin-bottom: auto;
    }

    h3 {
      font-size: clamp(2rem, 4vw, 3rem);
      font-weight: 700;
      margin-bottom: 1.5rem;
      color: #fff;
    }

    p {
      font-size: clamp(1rem, 1.5vw, 1.25rem);
      color: rgba(255, 255, 255, 0.6);
      line-height: 1.6;
      max-width: 80%;
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

    @media (max-width: 640px) {
      flex-direction: column;
      align-items: flex-start;
      gap: 1.5rem;
    }
  }

  &__header-left {
    display: flex;
    flex-direction: column;
    gap: 0.75rem;
  }

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

    @media (max-width: 640px) {
      align-items: flex-start;
    }
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

    .arrow {
      display: inline-block;
      transition: transform 0.3s ease;
    }

    &:hover {
      color: var(--color-accent);

      .arrow {
        transform: translate(3px, -3px);
      }
    }
  }

  &__grid {
    display: grid;
    grid-template-columns: 1.3fr 0.95fr 1.2fr;
    grid-template-rows: 38vh 38vh;
    gap: 1rem;

    @media (max-width: 992px) {
      grid-template-columns: 1fr 1fr;
      grid-template-rows: auto;
    }

    @media (max-width: 640px) {
      grid-template-columns: 1fr;
    }
  }

  &__item {
    background-color: #0e0e0e;
    clip-path: inset(100% 0% 0% 0% round 12px);
    will-change: clip-path;

    &--0 {
      grid-column: 1;
      grid-row: 1 / 3;
    }

    &--1 {
      grid-column: 2;
      grid-row: 1;
    }

    &--2 {
      grid-column: 2;
      grid-row: 2;
    }

    &--3 {
      grid-column: 3;
      grid-row: 1 / 3;
    }

    @media (max-width: 992px) {
      &--0, &--1, &--2, &--3 {
        grid-column: auto;
        grid-row: auto;
        aspect-ratio: 3 / 4;
      }
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

    &:hover img {
      transform: scale(1.14);
    }

    &:hover .gallery__overlay-content {
      transform: translateY(0);
      opacity: 1;
    }

    &:hover .gallery__index {
      opacity: 0;
    }
  }

  &__overlay {
    position: absolute;
    inset: 0;
    background: linear-gradient(
      to top,
      rgba(0, 0, 0, 0.88) 0%,
      rgba(0, 0, 0, 0.35) 45%,
      transparent 72%
    );
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

  &__meta {
    display: flex;
    justify-content: space-between;
    align-items: flex-end;
    gap: 0.5rem;
  }

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

  &__container {
    max-width: 1400px;
    margin: 0 auto;
  }

  &__header {
    margin-bottom: clamp(3rem, 10vh, 6rem);

    h2 {
      font-size: clamp(2.5rem, 6vw, 5rem);
      font-weight: 800;
      letter-spacing: -0.03em;
    }
  }

  &__layout {
    display: grid;
    grid-template-columns: 1fr 1.5fr;
    gap: clamp(4rem, 10vw, 8rem);

    @media (max-width: 992px) {
      grid-template-columns: 1fr;
      gap: 4rem;
    }
  }

  &__info {
    display: flex;
    flex-direction: column;
    gap: 3rem;

    .info-block {
      display: flex;
      flex-direction: column;
      gap: 0.5rem;

      .info-label {
        font-size: 0.8rem;
        text-transform: uppercase;
        letter-spacing: 0.2em;
        color: rgba(255, 255, 255, 0.4);
      }

      .info-value {
        font-size: clamp(1.2rem, 2vw, 1.8rem);
        font-weight: 500;
        color: #fff;
        text-decoration: none;
        transition: color 0.3s ease;

        &:hover {
          color: var(--color-accent);
        }
      }
    }

    .social-links {
      display: flex;
      gap: 2rem;
      flex-wrap: wrap;

      a {
        color: #fff;
        text-decoration: none;
        font-size: 1rem;
        opacity: 0.6;
        transition: opacity 0.3s ease, color 0.3s ease;

        &:hover {
          opacity: 1;
          color: var(--color-accent);
        }
      }
    }
  }

  &__form {
    display: flex;
    flex-direction: column;
    gap: 2.5rem;

    .form-group {
      position: relative;

      input,
      textarea {
        width: 100%;
        padding: 1rem 0;
        background: transparent;
        border: none;
        border-bottom: 1px solid rgba(255, 255, 255, 0.1);
        color: #fff;
        font-family: inherit;
        font-size: 1.1rem;
        transition: border-color 0.3s ease;

        &:focus {
          outline: none;
          border-color: var(--color-accent);
        }

        /* Float Label Logic */
        &:focus+label,
        &:not(:placeholder-shown)+label {
          transform: translateY(-20px);
          font-size: 0.8rem;
          color: var(--color-accent);
        }
      }

      label {
        position: absolute;
        left: 0;
        top: 1rem;
        color: rgba(255, 255, 255, 0.4);
        pointer-events: none;
        transition: all 0.3s ease;
      }
    }

    .submit-btn {
      align-self: flex-start;
      padding: 1.2rem 4rem;
      background: #fff;
      color: #000;
      border: none;
      border-radius: 100px;
      font-size: 1.1rem;
      font-weight: 600;
      cursor: pointer;
      position: relative;
      overflow: hidden;
      transition: transform 0.3s ease;

      span {
        position: relative;
        z-index: 2;
      }

      &:hover {
        transform: translateY(-5px);

        .btn-glow {
          opacity: 1;
        }
      }

      .btn-glow {
        position: absolute;
        inset: 0;
        background: var(--color-accent-gradient);
        opacity: 0;
        transition: opacity 0.3s ease;
        z-index: 1;
      }
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

    @media (max-width: 768px) {
      flex-direction: column;
      text-align: center;
    }
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

      &:hover {
        color: var(--color-accent);
      }
    }
  }
}
</style>
