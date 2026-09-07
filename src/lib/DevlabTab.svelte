<script>
  import PanelButton from './PanelButton.svelte';
</script>

<div class="devlab-tab">
  <p class="devlab-kicker">UNIT ELECTRONICS MX</p>
  <h1 class="text-xl font-bold">Devlab Laboratory</h1>
  <p class="text-sm text-slate-300">
    Entorno interactivo para experimentar, compilar y aprender con hardware
    abierto.
  </p>

  <div class="mt-3 flex flex-col gap-2">
    <PanelButton buttonIcon="fas fa-book-open" clickUrl="https://wiki.uelectronics.com/" buttonText="Abrir DevLab Wiki" />
    <PanelButton buttonIcon="fab fa-github" clickUrl="https://github.com/UNIT-Electronics-MX/" buttonText="Ecosistema en GitHub" />
    <PanelButton buttonIcon="fas fa-microchip" clickUrl="https://github.com/UNIT-Electronics-MX/unit_ch55x_sdk" buttonText="SDK CH55x" />
  </div>

  <div class="devlab-card mt-3">
    <p class="font-semibold">CH552 / SDCC</p>
    <p class="text-xs text-slate-300">Proyecto activo: examples/blink</p>
    <p class="mt-2 text-xs text-emerald-300">make · make hex · make bin</p>
  </div>
</div>
