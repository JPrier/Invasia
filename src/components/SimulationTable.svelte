<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  
  // Types for AI entities
  interface AiEntity {
    id: number;
    health: number;
    military_strength: number;
    position_x: number;
    position_y: number;
    state: number | string;
    territory: number;
    money: number;
    max_health: number;
  }

  // Purchase result type
  interface PurchaseResult {
    success: boolean;
    message: string;
  }

  // State names for display
  const stateNames = ['Idle', 'Active', 'Resting', 'Moving', 'Dead'];

  // Helper function to get state name
  function getStateName(state: number | string): string {
    if (typeof state === 'string') {
      return state;
    }
    return stateNames[state] || 'Unknown';
  }

  // Simulation control state
  let simulation: any = null;
  let wasmLoaded: boolean = false;
  let error: string | null = null;
  
  // Simulation data
  let entities: AiEntity[] = [];
  let tick: number = 0;
  let isRunning: boolean = false;
  
  // Configuration
  let entityCount: number = 100;
  let tickRate: number = 60;
  
  // Update loop
  let updateInterval: number | null = null;
  let renderInterval: number | null = null;

  // Purchase modal state
  let showPurchaseModal: boolean = false;
  let selectedEntityId: number | null = null;
  let selectedEntity: AiEntity | null = null;
  let purchaseType: string = 'military';
  let purchaseAmount: number = 10;
  let targetEntityId: number | null = null;
  let giftMoney: number = 0;
  let giftMilitary: number = 0;
  let giftTerritory: number = 0;
  let purchaseMessage: string = '';
  let purchaseSuccess: boolean = false;

  // Load the WASM module on component mount
  onMount(async () => {
    try {
      // Dynamically import the WASM module
      const wasmModule = await import('../wasm/wasm.js');
      await wasmModule.default();
      
      // Initialize the simulation
      simulation = new wasmModule.Simulation(entityCount);
      wasmLoaded = true;
      
      // Get initial snapshot
      updateSnapshot();
      
      // Start render loop (independent of simulation tick rate)
      startRenderLoop();
    } catch (e) {
      console.error('Failed to load WASM module:', e);
      error = 'Failed to load WebAssembly module';
      wasmLoaded = false;
    }
  });

  onDestroy(() => {
    stopSimulation();
    stopRenderLoop();
    if (simulation) {
      simulation.destroy();
    }
  });

  function startSimulation(): void {
    if (!simulation || !wasmLoaded) return;
    
    simulation.start();
    isRunning = true;
    
    // Update loop runs at tick rate
    const interval = 1000 / tickRate;
    updateInterval = window.setInterval(() => {
      simulation.update();
    }, interval);
  }

  function pauseSimulation(): void {
    if (!simulation || !wasmLoaded) return;
    
    simulation.pause();
    isRunning = false;
    
    if (updateInterval !== null) {
      clearInterval(updateInterval);
      updateInterval = null;
    }
  }

  function resumeSimulation(): void {
    if (!simulation || !wasmLoaded) return;
    
    if (isRunning) return; // Already running
    
    simulation.resume();
    isRunning = true;
    
    const interval = 1000 / tickRate;
    updateInterval = window.setInterval(() => {
      simulation.update();
    }, interval);
  }

  function stepSimulation(): void {
    if (!simulation || !wasmLoaded) return;
    simulation.step();
    updateSnapshot();
  }

  function resetSimulation(): void {
    if (!simulation || !wasmLoaded) return;
    
    pauseSimulation();
    simulation.reset();
    updateSnapshot();
  }

  function applyConfiguration(): void {
    if (!simulation || !wasmLoaded) return;
    
    pauseSimulation();
    simulation.set_entity_count(entityCount);
    simulation.set_tick_rate(tickRate);
    updateSnapshot();
  }

  function startRenderLoop(): void {
    // Render at ~30 FPS (independent of simulation tick rate)
    renderInterval = window.setInterval(() => {
      updateSnapshot();
    }, 1000 / 30);
  }

  function stopRenderLoop(): void {
    if (renderInterval !== null) {
      clearInterval(renderInterval);
      renderInterval = null;
    }
  }

  function stopSimulation(): void {
    if (updateInterval !== null) {
      clearInterval(updateInterval);
      updateInterval = null;
    }
    isRunning = false;
  }

  function updateSnapshot(): void {
    if (!simulation || !wasmLoaded) return;
    
    tick = simulation.get_tick();
    const snapshot = simulation.get_snapshot();
    entities = snapshot || [];
  }

  function formatNumber(num: number): string {
    return num.toFixed(2);
  }

  function openPurchaseModal(entity: AiEntity): void {
    selectedEntityId = entity.id;
    selectedEntity = entity;
    showPurchaseModal = true;
    purchaseMessage = '';
    purchaseAmount = 10;
    targetEntityId = null;
    giftMoney = 0;
    giftMilitary = 0;
    giftTerritory = 0;
  }

  function closePurchaseModal(): void {
    showPurchaseModal = false;
    selectedEntityId = null;
    selectedEntity = null;
    purchaseMessage = '';
  }

  function executePurchase(): void {
    if (!simulation || !wasmLoaded || selectedEntityId === null) return;
    
    let result: PurchaseResult;
    
    try {
      switch (purchaseType) {
        case 'military':
          result = simulation.purchase_military(selectedEntityId, purchaseAmount);
          break;
        case 'healing':
          result = simulation.purchase_healing(selectedEntityId, purchaseAmount);
          break;
        case 'maxHealth':
          result = simulation.purchase_max_health(selectedEntityId, purchaseAmount);
          break;
        case 'territory':
          if (targetEntityId === null) {
            purchaseMessage = 'Please select a target entity for territory trade';
            purchaseSuccess = false;
            return;
          }
          result = simulation.trade_territory(selectedEntityId, targetEntityId, purchaseAmount);
          break;
        case 'gift':
          if (targetEntityId === null) {
            purchaseMessage = 'Please select a target entity for gift';
            purchaseSuccess = false;
            return;
          }
          result = simulation.give_gift(selectedEntityId, targetEntityId, giftMoney, giftMilitary, giftTerritory);
          break;
        default:
          purchaseMessage = 'Unknown purchase type';
          purchaseSuccess = false;
          return;
      }
      
      purchaseSuccess = result.success;
      purchaseMessage = result.message;
      
      if (result.success) {
        updateSnapshot();
        // Close modal after successful purchase
        setTimeout(() => {
          closePurchaseModal();
        }, 1500);
      }
    } catch (e) {
      console.error('Purchase error:', e);
      purchaseMessage = 'Error executing purchase';
      purchaseSuccess = false;
    }
  }
</script>

<div class="simulation-container">
  <div class="header">
    <h2>AI Simulation</h2>
    {#if error}
      <p class="error">{error}</p>
    {/if}
    {#if wasmLoaded}
      <p class="wasm-badge">⚡ Powered by Rust + WebAssembly</p>
    {/if}
  </div>

  <div class="controls">
    <div class="control-group">
      <h3>Simulation Controls</h3>
      <div class="button-group">
        <button 
          on:click={startSimulation} 
          class="btn btn-success"
          disabled={!wasmLoaded || isRunning}
          aria-label="Start simulation"
        >
          ▶ Start
        </button>
        
        <button 
          on:click={pauseSimulation} 
          class="btn btn-warning"
          disabled={!wasmLoaded || !isRunning}
          aria-label="Pause simulation"
        >
          ⏸ Pause
        </button>
        
        <button 
          on:click={resumeSimulation} 
          class="btn btn-success"
          disabled={!wasmLoaded || isRunning}
          aria-label="Resume simulation"
        >
          ▶ Resume
        </button>
        
        <button 
          on:click={stepSimulation} 
          class="btn btn-primary"
          disabled={!wasmLoaded}
          aria-label="Step simulation"
        >
          ⏭ Step
        </button>
        
        <button 
          on:click={resetSimulation} 
          class="btn btn-danger"
          disabled={!wasmLoaded}
          aria-label="Reset simulation"
        >
          ⏹ Reset
        </button>
      </div>
    </div>

    <div class="control-group">
      <h3>Configuration</h3>
      <div class="config-inputs">
        <label>
          Entity Count: <span>{entityCount}</span>
          <input 
            type="range" 
            min="10" 
            max="1000" 
            step="10"
            bind:value={entityCount}
          />
        </label>
        
        <label>
          Tick Rate: <span>{tickRate} Hz</span>
          <input 
            type="range" 
            min="1" 
            max="120" 
            step="1"
            bind:value={tickRate}
          />
        </label>
        
        <button 
          on:click={applyConfiguration} 
          class="btn btn-primary"
          disabled={!wasmLoaded}
        >
          Apply Config
        </button>
      </div>
    </div>

    <div class="stats">
      <p><strong>Tick:</strong> {tick}</p>
      <p><strong>Entities:</strong> {entities.length}</p>
      <p><strong>Status:</strong> {isRunning ? '🟢 Running' : '🔴 Paused'}</p>
    </div>
  </div>

  <div class="table-container">
    <table class="entity-table">
      <thead>
        <tr>
          <th>ID</th>
          <th>Health</th>
          <th>Military Strength</th>
          <th>Money</th>
          <th>Territory</th>
          <th>Position X</th>
          <th>Position Y</th>
          <th>State</th>
          <th>Actions</th>
        </tr>
      </thead>
      <tbody>
        {#each entities as entity (entity.id)}
          <tr>
            <td>{entity.id}</td>
            <td class="health-cell">
              <div class="bar-container">
                <div class="bar bar-health" style="width: {entity.health}%"></div>
                <span class="bar-text">{formatNumber(entity.health)}</span>
              </div>
            </td>
            <td class="military-cell">
              <div class="bar-container">
                <div class="bar bar-military" style="width: {entity.military_strength}%"></div>
                <span class="bar-text">{formatNumber(entity.military_strength)}</span>
              </div>
            </td>
            <td class="money-cell">
              <div class="bar-container">
                <div class="bar bar-money" style="width: {Math.min(entity.money / 2, 100)}%"></div>
                <span class="bar-text">{formatNumber(entity.money)}</span>
              </div>
            </td>
            <td class="territory-cell">
              <div class="bar-container">
                <div class="bar bar-territory" style="width: {entity.territory}%"></div>
                <span class="bar-text">{formatNumber(entity.territory)}</span>
              </div>
            </td>
            <td>{formatNumber(entity.position_x)}</td>
            <td>{formatNumber(entity.position_y)}</td>
            <td class="state-cell">
              <span class="state-badge state-{typeof entity.state === 'number' ? entity.state : 0}">
                {getStateName(entity.state)}
              </span>
            </td>
            <td class="actions-cell">
              <button 
                class="btn-action" 
                on:click={() => openPurchaseModal(entity)}
                disabled={entity.state === 4}
                aria-label="Purchase options for entity {entity.id}"
              >
                💰 Purchase
              </button>
            </td>
          </tr>
        {/each}
      </tbody>
    </table>
  </div>

  {#if showPurchaseModal && selectedEntity}
    <div class="modal-overlay" on:click={closePurchaseModal}>
      <div class="modal-content" on:click|stopPropagation>
        <h3>Purchase Options - Entity #{selectedEntity.id}</h3>
        
        <div class="entity-info">
          <p><strong>Money:</strong> {formatNumber(selectedEntity.money)}</p>
          <p><strong>Health:</strong> {formatNumber(selectedEntity.health)} / {formatNumber(selectedEntity.max_health)}</p>
          <p><strong>Military:</strong> {formatNumber(selectedEntity.military_strength)}</p>
          <p><strong>Territory:</strong> {formatNumber(selectedEntity.territory)}</p>
        </div>

        <div class="purchase-form">
          <label>
            <strong>Purchase Type:</strong>
            <select bind:value={purchaseType}>
              <option value="military">Military Strength (Cost: 10 per unit)</option>
              <option value="healing">Healing/Repairs (Cost: 5 per unit)</option>
              <option value="maxHealth">Increase Max Health (Cost: 20 per unit)</option>
              <option value="territory">Trade Territory (Cost: 15 per unit)</option>
              <option value="gift">Give Gift (Free)</option>
            </select>
          </label>

          {#if purchaseType !== 'gift'}
            <label>
              <strong>Amount:</strong>
              <input type="number" bind:value={purchaseAmount} min="1" max="100" step="1" />
            </label>
          {/if}

          {#if purchaseType === 'territory' || purchaseType === 'gift'}
            <label>
              <strong>Target Entity ID:</strong>
              <input type="number" bind:value={targetEntityId} min="0" max={entityCount - 1} step="1" />
            </label>
          {/if}

          {#if purchaseType === 'gift'}
            <label>
              <strong>Money to Give:</strong>
              <input type="number" bind:value={giftMoney} min="0" max={selectedEntity.money} step="1" />
            </label>
            <label>
              <strong>Military to Give:</strong>
              <input type="number" bind:value={giftMilitary} min="0" max={selectedEntity.military_strength} step="1" />
            </label>
            <label>
              <strong>Territory to Give:</strong>
              <input type="number" bind:value={giftTerritory} min="0" max={selectedEntity.territory} step="1" />
            </label>
          {/if}

          {#if purchaseType === 'military'}
            <p class="cost-info">Cost: {formatNumber(purchaseAmount * 10)} money</p>
          {:else if purchaseType === 'healing'}
            <p class="cost-info">Cost: {formatNumber(purchaseAmount * 5)} money</p>
          {:else if purchaseType === 'maxHealth'}
            <p class="cost-info">Cost: {formatNumber(purchaseAmount * 20)} money</p>
          {:else if purchaseType === 'territory'}
            <p class="cost-info">Cost: {formatNumber(purchaseAmount * 15)} money</p>
          {/if}

          {#if purchaseMessage}
            <div class="purchase-message {purchaseSuccess ? 'success' : 'error'}">
              {purchaseMessage}
            </div>
          {/if}

          <div class="modal-actions">
            <button class="btn btn-primary" on:click={executePurchase}>
              {purchaseType === 'gift' ? 'Give Gift' : 'Purchase'}
            </button>
            <button class="btn btn-danger" on:click={closePurchaseModal}>
              Close
            </button>
          </div>
        </div>
      </div>
    </div>
  {/if}
</div>

<style>
  .simulation-container {
    width: 100%;
    max-width: 1400px;
    margin: 0 auto;
  }

  .header {
    text-align: center;
    margin-bottom: 1.5rem;
  }

  .header h2 {
    margin: 0 0 0.5rem 0;
    color: #333;
  }

  .wasm-badge {
    font-size: 0.875rem;
    color: #10b981;
    font-weight: 600;
    margin: 0.5rem 0;
  }

  .error {
    color: #ef4444;
    font-size: 0.875rem;
    margin: 0.5rem 0;
  }

  .controls {
    background: #f5f5f5;
    border-radius: 0.5rem;
    padding: 1rem;
    margin-bottom: 1rem;
  }

  .control-group {
    margin-bottom: 1rem;
  }

  .control-group:last-child {
    margin-bottom: 0;
  }

  .control-group h3 {
    margin: 0 0 0.5rem 0;
    font-size: 1rem;
    color: #555;
  }

  .button-group {
    display: flex;
    gap: 0.5rem;
    flex-wrap: wrap;
  }

  .config-inputs {
    display: flex;
    flex-direction: column;
    gap: 0.75rem;
  }

  .config-inputs label {
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
    font-size: 0.875rem;
    color: #555;
  }

  .config-inputs label span {
    font-weight: 600;
    color: #333;
  }

  .config-inputs input[type="range"] {
    width: 100%;
  }

  .stats {
    display: flex;
    gap: 2rem;
    margin-top: 1rem;
    padding-top: 1rem;
    border-top: 1px solid #ddd;
  }

  .stats p {
    margin: 0;
    font-size: 0.875rem;
    color: #555;
  }

  .btn {
    padding: 0.5rem 1rem;
    font-size: 0.875rem;
    font-weight: 500;
    color: white;
    border: none;
    border-radius: 0.375rem;
    cursor: pointer;
    transition: all 0.2s;
    box-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
  }

  .btn:hover:not(:disabled) {
    transform: translateY(-1px);
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.15);
  }

  .btn:active:not(:disabled) {
    transform: scale(0.98);
  }

  .btn:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  .btn-primary {
    background: #667eea;
  }

  .btn-primary:hover:not(:disabled) {
    background: #5568d3;
  }

  .btn-success {
    background: #10b981;
  }

  .btn-success:hover:not(:disabled) {
    background: #059669;
  }

  .btn-warning {
    background: #f59e0b;
  }

  .btn-warning:hover:not(:disabled) {
    background: #d97706;
  }

  .btn-danger {
    background: #ef4444;
  }

  .btn-danger:hover:not(:disabled) {
    background: #dc2626;
  }

  .table-container {
    overflow: auto;
    max-height: 600px;
    border: 1px solid #ddd;
    border-radius: 0.5rem;
    background: white;
  }

  .entity-table {
    width: 100%;
    border-collapse: collapse;
    font-size: 0.875rem;
  }

  .entity-table thead {
    position: sticky;
    top: 0;
    background: #667eea;
    color: white;
    z-index: 10;
  }

  .entity-table th {
    padding: 0.75rem;
    text-align: left;
    font-weight: 600;
    border-bottom: 2px solid #5568d3;
  }

  .entity-table tbody tr {
    border-bottom: 1px solid #e5e7eb;
  }

  .entity-table tbody tr:hover {
    background-color: #f9fafb;
  }

  .entity-table td {
    padding: 0.5rem 0.75rem;
  }

  .bar-container {
    position: relative;
    width: 100%;
    height: 20px;
    background: #e5e7eb;
    border-radius: 0.25rem;
    overflow: hidden;
  }

  .bar {
    position: absolute;
    top: 0;
    left: 0;
    height: 100%;
    border-radius: 0.25rem;
    transition: width 0.3s ease;
  }

  .bar-health {
    background: linear-gradient(90deg, #ef4444 0%, #10b981 100%);
  }

  .bar-military {
    background: linear-gradient(90deg, #6366f1 0%, #8b5cf6 100%);
  }

  .bar-territory {
    background: linear-gradient(90deg, #eab308 0%, #22c55e 100%);
  }

  .bar-money {
    background: linear-gradient(90deg, #f59e0b 0%, #10b981 100%);
  }

  .bar-text {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    font-size: 0.75rem;
    font-weight: 600;
    color: #333;
    z-index: 1;
    text-shadow: 0 0 2px white;
  }

  .state-badge {
    display: inline-block;
    padding: 0.25rem 0.5rem;
    border-radius: 0.25rem;
    font-size: 0.75rem;
    font-weight: 600;
    text-align: center;
  }

  .state-0 {
    background: #e5e7eb;
    color: #6b7280;
  }

  .state-1 {
    background: #dbeafe;
    color: #1e40af;
  }

  .state-2 {
    background: #fef3c7;
    color: #92400e;
  }

  .state-3 {
    background: #d1fae5;
    color: #065f46;
  }

  .state-4 {
    background: #fee2e2;
    color: #991b1b;
  }

  .actions-cell {
    text-align: center;
  }

  .btn-action {
    padding: 0.25rem 0.5rem;
    font-size: 0.75rem;
    font-weight: 500;
    color: white;
    background: #667eea;
    border: none;
    border-radius: 0.25rem;
    cursor: pointer;
    transition: all 0.2s;
  }

  .btn-action:hover:not(:disabled) {
    background: #5568d3;
  }

  .btn-action:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  .modal-overlay {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: rgba(0, 0, 0, 0.5);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000;
  }

  .modal-content {
    background: white;
    border-radius: 0.5rem;
    padding: 2rem;
    max-width: 500px;
    width: 90%;
    max-height: 90vh;
    overflow-y: auto;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  }

  .modal-content h3 {
    margin: 0 0 1rem 0;
    color: #333;
  }

  .entity-info {
    background: #f5f5f5;
    border-radius: 0.375rem;
    padding: 0.75rem;
    margin-bottom: 1rem;
  }

  .entity-info p {
    margin: 0.25rem 0;
    font-size: 0.875rem;
  }

  .purchase-form {
    display: flex;
    flex-direction: column;
    gap: 1rem;
  }

  .purchase-form label {
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
    font-size: 0.875rem;
  }

  .purchase-form input,
  .purchase-form select {
    padding: 0.5rem;
    border: 1px solid #ddd;
    border-radius: 0.25rem;
    font-size: 0.875rem;
  }

  .cost-info {
    background: #fef3c7;
    padding: 0.5rem;
    border-radius: 0.25rem;
    font-size: 0.875rem;
    font-weight: 600;
    color: #92400e;
    margin: 0;
  }

  .purchase-message {
    padding: 0.75rem;
    border-radius: 0.25rem;
    font-size: 0.875rem;
    font-weight: 500;
  }

  .purchase-message.success {
    background: #d1fae5;
    color: #065f46;
  }

  .purchase-message.error {
    background: #fee2e2;
    color: #991b1b;
  }

  .modal-actions {
    display: flex;
    gap: 0.5rem;
    margin-top: 1rem;
  }

  .modal-actions button {
    flex: 1;
  }

  @media (max-width: 768px) {
    .button-group {
      flex-direction: column;
    }

    .btn {
      width: 100%;
    }

    .stats {
      flex-direction: column;
      gap: 0.5rem;
    }

    .table-container {
      max-height: 400px;
    }

    .modal-content {
      width: 95%;
      padding: 1rem;
    }
  }
</style>
