<script>
  import { onMount } from 'svelte';
  import { auth } from "../stores/auth";
  import { goto } from '$app/navigation';
  import { initFirebase } from "$lib/client/firebase";
  import { doc, getDoc, setDoc } from "firebase/firestore";
  let isLoading = true;
  let username = '';
  const { db } = initFirebase();
  let userData = {
      bmi: 0,
      bodyFatPercentage: 0,
      ffm: 0,
      tbw: 0,
      bmr: 0
  };
  let isDarkMode = false;

  // Check for saved theme preference
  onMount(() => {
    const savedTheme = localStorage.getItem('theme');
    if (savedTheme) {
      isDarkMode = savedTheme === 'dark';
      document.documentElement.setAttribute('data-theme', savedTheme);
    } else {
      const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
      isDarkMode = prefersDark;
      document.documentElement.setAttribute('data-theme', isDarkMode ? 'dark' : 'light');
    }
  });

  function toggleTheme() {
    isDarkMode = !isDarkMode;
    const theme = isDarkMode ? 'dark' : 'light';
    document.documentElement.setAttribute('data-theme', theme);
    localStorage.setItem('theme', theme);
  }

  // Calculation functions
  function calculateBMI(weight, height) {
      return weight / (height * height);
  }

  function calculateBodyFatPercentageMale(weight, impedance) {
      const bfMass = -10.463 + (0.671 * weight) - (0.184385 * impedance);
      return (bfMass / weight) * 100;
  }

  function calculateBodyFatPercentageFemale(weight, impedance) {
      const bfMass = -9.411 + (0.518 * weight) - (0.206987 * impedance);
      return (bfMass / weight) * 100;
  }

  function calculateFFM(weight, bodyFatPercentage) {
      return weight * (1 - (bodyFatPercentage / 100));
  }

  function calculateTBW(weight, height, impedance, gender) {
      const factor = (gender === 'M') ? 0.155 : 0.245;
      return (2.447 - (0.09156 * impedance) + (0.1074 * height)) * weight * factor;
  }

  function calculateBMR(weight, height, age, gender) {
      if (gender === 'M') {
          return (10 * weight) + (6.25 * height * 100) - (5 * age) + 5;
      }
      return (10 * weight) + (6.25 * height * 100) - (5 * age) - 161;
  }

  const handleLogout = async () => {
      try {
          await auth.signOut();
          goto('/login');
      } catch (error) {
          console.error('Error logging out:', error);
      }
  };

  onMount(() => {
      setTimeout(() => {
          isLoading = false;
      }, 1000);
      
      return auth.subscribe(async (user) => {
          if (!user) {
              goto('/login');
          } else {
              username = user.displayName || user.email?.split('@')[0] || 'User';
              
              try {
                  const userDocRef = doc(db, "userdata", user.email);
                  const userDoc = await getDoc(userDocRef);
                  if (!userDoc.exists() || !userDoc.data().cred) {
                      goto('/creds');
                  } else {
                      const data = userDoc.data();
                      if (data.metrics) {
                          // Use stored metrics if available
                          userData = data.metrics;
                      } else {
                          const cred = data.cred;
                          const height = parseFloat(cred.height) / 100; // Convert to meters
                          const weight = parseFloat(cred.weight);
                          const impedance = parseFloat(cred.impedance);
                          const age = parseInt(cred.age);
                          const gender = cred.gender;

                          const bodyFatPercentage = gender === 'M' 
                              ? calculateBodyFatPercentageMale(weight, impedance)
                              : calculateBodyFatPercentageFemale(weight, impedance);

                          userData = {
                              bmi: calculateBMI(weight, height),
                              bodyFatPercentage: bodyFatPercentage,
                              ffm: calculateFFM(weight, bodyFatPercentage),
                              tbw: calculateTBW(weight, height, impedance, gender),
                              bmr: calculateBMR(weight, height, age, gender)
                          };

                          // Store calculated metrics in Firebase
                          await setDoc(userDocRef, { metrics: userData }, { merge: true });
                      }
                  }
              } catch (error) {
                  console.error("Error checking user credentials:", error);
                  goto('/creds');
              }
          }
      });
  });

  $: stats = [
      { title: 'FFM', value: userData.ffm.toFixed(1) },
      { title: 'BMR', value: userData.bmr.toFixed(1) },
      { title: 'TBW', value: userData.tbw.toFixed(1) }
  ];
</script>

{#if isLoading}
<div class="loading-spinner"></div>
{:else}
<div class="min-h-screen" style="background-color: var(--bg-color); color: var(--text-color);">
  <div class="p-4 md:p-6 lg:p-8">
    <nav class="flex flex-wrap gap-4 mb-12 animate-fade-in">
      <button 
        class="absolute top-4 right-4 z-50 p-3 rounded-full shadow-lg"
        style="background-color: var(--button-bg-color); color: var(--button-text-color);"
        on:click={toggleTheme}
        in:fade={{ duration: 800, delay: 1000 }}
      >
        {#if isDarkMode}
          <img src="/icons/moon.svg" alt="Dark mode" class="w-6 h-6" />
        {:else}
          <img src="/icons/sun.svg" alt="Light mode" class="w-6 h-6" />
        {/if}
      </button>

      <button 
        on:click={() => goto('/')}
        class="p-3 rounded-full shadow-lg"
        style="background-color: var(--button-bg-color); color: var(--button-text-color);"
      >
        <img src="/icons/dash.svg" alt="Dashboard" class="w-6 h-6" />
      </button>
      <button 
        on:click={() => goto('/dietplan')}
        class="p-3 rounded-full shadow-lg"
        style="background-color: var(--button-bg-color); color: var(--button-text-color);"
      >
        <img src="/icons/diet.svg" alt="Diet Plan" class="w-6 h-6" />
      </button>
      <button 
        on:click={() => goto('/creds')}
        class="p-3 rounded-full shadow-lg"
        style="background-color: var(--button-bg-color); color: var(--button-text-color);"
      >
        <img src="/icons/add.svg" alt="Add New Values" class="w-6 h-6" />
      </button>
      <button 
        on:click={handleLogout}
        class="p-3 rounded-full shadow-lg"
        style="background-color: var(--logout-bg-color); color: var(--logout-text-color);"
      >
        <img src="/icons/close.svg" alt="Logout" class="w-6 h-6" />
      </button>
    </nav>

    <header class="mb-12 animate-fade-in animation-delay-200">
      <h1 class="text-4xl font-medium mb-2" style="color: var(--text-color);">Overview</h1>
      <h2 class="text-4xl font-normal" style="color: var(--text-color);">Patient Health</h2>
    </header>

    <div class="md:relative h-[70vh] flex flex-col md:flex-row gap-8">
      <div class="flex justify-center items-center w-full md:w-1/3 max-w-[500px] min-w-[200px] mx-auto animate-fade-in animation-delay-300">
        <script type="module" src="https://unpkg.com/@splinetool/viewer@1.9.48/build/spline-viewer.js"></script>
        <spline-viewer width="100%" height="700" loading-anim-type="spinner-small-dark" url="https://prod.spline.design/ieC6db2R-ndItLa7/scene.splinecode"></spline-viewer>
      </div>

      <main class="grid grid-cols-1 w-full md:grid-cols-2 lg:grid-cols-2 gap-8 flex-grow max-w-[1200px]">
        <!-- BMI Card -->
        <div class="p-8 rounded-3xl shadow-lg hover:shadow-xl transition-shadow animate-slide-up animation-delay-400" style="background-color: var(--bmi-bg-color); color: var(--bmi-text-color);">
          <p class="text-xl opacity-90 mb-8 leading-relaxed">
            Your current BMI is {userData.bmi < 18.5 ? 'below normal' : userData.bmi < 25 ? 'very good' : userData.bmi < 30 ? 'above normal' : 'high'}
          </p>
          <div class="text-6xl font-medium" style="color: {userData.bmi < 25 ? 'green' : 'yellow'};">
              {userData.bmi.toFixed(1)}
          </div>
        </div>

        <!-- Fat Percentage Card -->
        <div class="p-8 rounded-3xl shadow-lg hover:shadow-xl transition-shadow animate-slide-up animation-delay-500" style="background-color: var(--card-bg-color); color: var(--card-text-color);">
          <p class="text-xl mb-4">Total Fat %</p>
          <div class="text-6xl font-medium">{userData.bodyFatPercentage.toFixed(1)}%</div>
        </div>

        <!-- Stats Grid -->
        <div class="flex flex-col md:flex-row gap-4 w-full col-span-full">
          {#each stats as stat, i}
            <div class="flex-1 p-7 rounded-3xl shadow-lg hover:shadow-xl transition-shadow animate-slide-up" 
                 style="animation-delay: {600 + (i * 100)}ms; background-color: var(--card-bg-color); color: var(--card-text-color);">
              <h3 class="text-2xl font-medium mb-2">{stat.title}</h3>
              <p class="mb-3">your {stat.title} is</p>
              <div class="text-4xl font-medium">{stat.value}</div>
            </div>
          {/each}
        </div>
      </main>
    </div>
  </div>
</div>
{/if}

<style>
/* Loading Spinner */
.loading-spinner {
  border: 4px solid #f3f3f3;
  border-top: 4px solid #4867AA;
  border-radius: 50%;
  width: 40px;
  height: 40px;
  animation: spin 1s linear infinite;
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}

@keyframes spin {
  0% { transform: translate(-50%, -50%) rotate(0deg); }
  100% { transform: translate(-50%, -50%) rotate(360deg); }
}

/* Fade In Animation */
@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

/* Slide Up Animation */
@keyframes slideUp {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* Animation Classes */
.animate-fade-in {
  opacity: 0;
  animation: fadeIn 0.6s ease-out forwards;
}

.animate-slide-up {
  opacity: 0;
  animation: slideUp 0.6s ease-out forwards;
}

/* Animation Delays */
.animation-delay-200 { animation-delay: 200ms; }
.animation-delay-300 { animation-delay: 300ms; }
.animation-delay-400 { animation-delay: 400ms; }
.animation-delay-500 { animation-delay: 500ms; }

@keyframes float {
  0%, 100% { transform: translateY(0px); }
  50% { transform: translateY(-20px); }
}

/* Highlight style for the active button */
.highlight {
  border: 2px solid var(--highlight-border-color);
  background-color: var(--highlight-bg-color);
  color: var(--highlight-text-color);
}

/* Theme-specific styles */
:root {
  --highlight-border-color: black; /* Default for light mode */
}

[data-theme='dark'] {
  --highlight-border-color: white;
}

:root {
  --button-bg-color: #f3f3f3;
  --button-text-color: #333;
  --logout-bg-color: #ff4d4d;
  --logout-text-color: #fff;
}

[data-theme='dark'] {
  --button-bg-color: #333;
  --button-text-color: #f3f3f3;
  --logout-bg-color: #ff4d4d;
  --logout-text-color: #fff;
}

/* Mobile-specific styles */
@media (max-width: 768px) {
  nav button {
    width: 56px;
    height: 56px;
  }
}
</style>