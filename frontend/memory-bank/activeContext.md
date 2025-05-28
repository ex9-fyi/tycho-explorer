# Active Context

## Current Work Focus

### 🚨 CRITICAL FIX - Graph View Layout Preservation (2025-05-27)

**PROBLEM**: Graph view was re-centering on every block update, destroying user's zoom, pan, and node manipulations.

**ROOT CAUSE**: Multiple issues causing unwanted re-centering:
1. Mobile-friendly code added `network.fit()` calls in updateData method
2. Initial useEffect was re-initializing graph on data changes
3. `autoResize: true` was causing automatic viewport adjustments
4. vis-network's internal layout recalculations on data updates

**SOLUTION IMPLEMENTED**:
1. **Removed ALL fit() calls** - No auto-fitting anywhere in the code
2. **Set autoResize: false** - Prevents automatic resizing/re-centering
3. **Separated initialization** - Graph manager created once, data updates don't reinitialize
4. **Viewport preservation** - Save and restore viewport position/scale during updates:
   ```typescript
   // In updateData method:
   const currentViewPosition = this.network.getViewPosition();
   const currentScale = this.network.getScale();
   // ... perform updates ...
   this.network.moveTo({
     position: currentViewPosition,
     scale: currentScale,
     animation: false
   });
   ```

**CRITICAL**: This fix MUST be preserved. Users expect their graph manipulations to persist.

### 🚨 CRITICAL FIX - Mobile Edge Tooltip Touch Handler (2025-05-27)

**PROBLEM**: Edge tooltips stopped working on mobile after the layout preservation fix.

**ROOT CAUSE**: The touchend event listener was only added during initialization. Since initialization now runs only once (to prevent re-centering), if the graph initialized on desktop and then viewed on mobile, the touch handler wasn't present.

**SOLUTION IMPLEMENTED**:
1. **Made GraphManager properties accessible**:
   - Changed `network` from private to public
   - Changed `showEdgeInfoPopover` from private to public
2. **Added separate useEffect for mobile touch handling**:
   ```typescript
   useEffect(() => {
     if (!containerRef.current || !graphManagerRef.current || !graphManagerRef.current.isInitialized() || !isMobile) return;
     
     const canvas = containerRef.current.querySelector('canvas');
     if (!canvas) return;
     
     const handleTouchEnd = (event: TouchEvent) => {
       event.preventDefault();
       const touch = event.changedTouches[0];
       const manager = graphManagerRef.current;
       if (!manager || !manager.network) return;
       
       const canvasPos = manager.network.DOMtoCanvas({ 
         x: touch.clientX, 
         y: touch.clientY 
       });
       const edgeId = manager.network.getEdgeAt(canvasPos);
       
       if (edgeId) {
         manager.showEdgeInfoPopover(String(edgeId), touch.clientX, touch.clientY);
       }
     };
     
     canvas.addEventListener('touchend', handleTouchEnd, { passive: false });
     
     return () => {
       canvas.removeEventListener('touchend', handleTouchEnd);
     };
   }, [isMobile, tokenNodes]);
   ```

**CRITICAL**: This ensures edge tooltips work on mobile regardless of initialization order.

### Recent Development Updates (2025-05-28 - Latest Session)

#### UI Consistency Improvements
1. **Token Filter Sorting Consistency**:
   - **Issue**: List View token filter didn't sort selected tokens first, unlike Graph View
   - **Solution**: Updated `PoolListFilterBar.tsx` filteredTokens logic to match Graph View behavior
   - **Implementation**:
     ```typescript
     // In PoolListFilterBar.tsx
     const filteredTokens = useMemo(() => {
       // First filter tokens based on search
       const filtered = allTokensForFilter.filter(token => /* search logic */);
       
       // Then sort to put selected tokens first
       return filtered.sort((a, b) => {
         const aIsSelected = selectedTokens.some(st => st.address === a.address);
         const bIsSelected = selectedTokens.some(st => st.address === b.address);
         
         if (aIsSelected && !bIsSelected) return -1; // a comes first
         if (!aIsSelected && bIsSelected) return 1; // b comes first
         // If both are selected or both are unselected, sort by symbol
         return a.symbol.localeCompare(b.symbol);
       });
     }, [allTokensForFilter, tokenSearch, selectedTokens]);
     ```
   - **Critical**: This pattern should be preserved for UI consistency across all views

2. **Environment Variables Configuration**:
   - Added WebSocket URL configuration via environment variables
   - Created `.env` and `.env.example` files
   - Priority: localStorage > env variable > hardcoded default
   - Updated TypeScript definitions in `vite-env.d.ts`
   - Context always shows current active URL for transparency

3. **Mobile Sidebar Styling**:
   - Pool detail sidebar now has glass-like opaque background on mobile
   - Mobile: `bg-[rgba(20,10,35,0.98)] backdrop-blur-[104px]`
   - Desktop: `bg-[rgba(255,255,255,0.01)] backdrop-blur-[200px]`
   - Maintains readability while preserving visual aesthetics

### Recent Development Updates (2025-05-27 - Latest Session - Mobile Friendliness)

#### Mobile-Friendly Implementation (Completed)
1. **Graph View Touch Support**:
   - Enabled touch interactions in vis-network with proper configuration
   - Added useIsMobile hook integration for mobile detection
   - Implemented mobile-optimized physics parameters:
     - Reduced gravitational constant from -25000 to -15000 for tighter clustering
     - Increased central gravity from 0.1 to 0.3 to keep nodes in view
     - Shortened spring length from 300 to 150 for compact layout
   - Enhanced auto-centering with animation on mobile devices
   - Added re-centering when new nodes are added during data updates
   - **Fixed node tooltips** not showing on click by correcting networkOptions reference
   - **Fixed edge tooltips** on mobile - now respond to single tap instead of requiring long press
     - Added direct touchend event handler for immediate response
     - Properly converts DOM to canvas coordinates for accurate edge detection

2. **Responsive Header Layout**:
   - Converted header from single row to responsive 2-row layout on mobile
   - Mobile layout: Brand/Network on top row, View selector on bottom row
   - Desktop maintains original single-row layout
   - Responsive gaps: gap-2 on mobile, gap-4 on desktop
   - Text/icon sizing adjusts for mobile screens

3. **Mobile Touch Targets**:
   - All interactive elements now meet 44px minimum touch target
   - Filter buttons: h-10 on mobile, h-8 on desktop
   - ViewSelector tabs: py-3 px-4 on mobile for better touch
   - App menu selector: w-11 h-11 on mobile

4. **Responsive Filter Controls**:
   - Filter controls wrap on mobile screens
   - Full-width buttons on mobile for easier interaction
   - Block progress indicator moves to right on mobile
   - Graph controls use full width on small screens
   - **Filter popovers truncate** long lists to prevent screen overflow:
     - Tokens show first 3 items then "..."
     - Protocols show first 2 items then "..."

5. **TypeScript Type Fixes**:
   - Resolved all type errors in GraphView.tsx
   - Used VisNode/VisEdge aliases to avoid conflicts with browser's Node type
   - Proper type imports from vis-network library

6. **Mobile Table Scrolling Fix**:
   - Fixed touch scrolling issue in Pool List View table that was affecting Chrome Android
   - **Solution**: Replaced table structure with div-based layout on mobile to avoid Chrome bugs
   - Removed sticky table headers on mobile (Chrome Android bug with sticky + overflow)
   - Added Chrome-specific fixes:
     - `display: 'block'` for proper block formatting context
     - `position: 'relative'` for positioning context
     - Simpler overflow structure without nested containers
   - Maintained infinite scroll functionality on both platforms
   - **Result**: Touch scrolling now works perfectly on both Chrome and Firefox mobile browsers

### Recent Development Updates (2025-05-27 - Earlier Session)

#### Swap Simulator Number Display
1. **Output Amount Display**:
   - Limited to maximum 9 decimal places for cleaner display
   - Uses smart formatting: shows actual decimals up to 9 places
   - Example: "274.499327076" displays as "274.499327076" (not padded with zeros)
   - If value has fewer decimals, shows only those (e.g., "100.5" stays "100.5")
   
2. **Net Amount Precision**:
   - Always displays full precision - never truncated
   - Critical for users to see exact amounts they'll receive
   - Maintains consistency with user requirement

3. **Bug Fix - TypeError**:
   - Fixed "amount.split is not a function" error in SwapSimulator
   - Issue: `amount` was a number but `.split()` was called on it
   - Solution: Convert to string first with `String(amount).split()`
   - Error occurred at line 121 in SwapCard component

### Recent Development Updates (2025-05-27)

#### Graph View UI Enhancements
1. **Graph Pan Controls Implementation**:
   - Created comprehensive PanManager class for Figma-like pan controls
   - Middle mouse button drag for panning (like Figma)
   - Two-finger trackpad pan support
   - Disabled default left-click pan in vis-network (`dragView: false`)
   - Used requestAnimationFrame for smooth 60fps panning
   - Proper event handling and cleanup in GraphManager

2. **App Menu Selector**:
   - Added app menu selector in header (left of Tycho Explorer logo)
   - Downloads grid dots icon from Figma assets
   - Dropdown shows Explorer (current app, disabled) and Orderbook (links to orderbook.wtf)
   - Glassy styling: `bg-[rgba(255,244,224,0.04)]` with backdrop blur
   - Folly red border (#FF3366) for dropdown

3. **Graph View Background Restoration**:
   - Analyzed Figma design layers to understand purple effect
   - Purple artboard at 25% opacity (reduced from 40% to let rays through)
   - Added noise texture layer at 10% opacity
   - Removed opaque `bg-[#190A35]` to transparent background
   - Reduced backdrop blur from 24px to 16px for more detail
   - Base background reduced to 0.01 opacity

4. **Node Color Fixes**:
   - Restored warm cream node colors from dark colors
   - Background: `rgba(255, 244, 224, 0.04)` → `rgba(255, 244, 224, 0.08)` on highlight
   - Text color: `rgba(255, 244, 224, 1)`
   - Hover states with warm cream at 0.06 opacity

5. **WebSocket Config Restyling**:
   - Updated all form elements to glassy style
   - Select dropdown: transparent borders, warm cream text
   - Input field and button matching app selector style
   - Height increased to h-10 for consistency

6. **Filter Search Bars Update**:
   - Matched GraphControls dynamic border implementation
   - Wrapper div with conditional border styling
   - Default: 1px `rgba(255, 244, 224, 0.2)` border
   - Focused: 2px `#FF3366` (folly red) border
   - Added focus state tracking with separate state variables

### Recent Development Updates (2025-05-27)

#### UI Refinements and Fixes
1. **Token Address Formatting**:
   - List View now formats raw address tokens (e.g., 0x9f8f72aa9304c8b593d555f12ef6589cc3a579a2) as shortened format (0x9f8f...79a2)
   - Swap Simulator also formats address tokens consistently
   - Uses existing `renderHexId` function for consistency

2. **Filter Popover Styling**:
   - Updated all popovers to match Figma design with transparent borders
   - Background: `rgba(255,244,224,0.02)` for depth 5 transparency
   - Border: `rgba(255,244,224,0.12)` for warm transparent effect
   - Checkboxes use folly red (#FF3366) fill with white checkmark
   - Filter buttons have dashed borders with transparent styling

3. **Favicon Configuration**:
   - Fixed empty favicon.ico issue
   - Configured to use /public/favicon.ico with proper MIME type
   - Changed from `image/svg+xml` to `image/x-icon` in index.html

4. **Swap Simulator Precision**:
   - Removed truncation of net amount - now shows full precision
   - Buy/Sell amounts display at full precision (no toFixed(2))
   - Users see exact values from simulation API

#### Graph View Edge Widening Investigation
1. **Root Cause Analysis**: 
   - Added comprehensive logging throughout the data flow (WebSocket → PoolDataContext → useGraphData → vis-network)
   - Discovered WebSocket sends initial pool data but subsequent block updates contain 0 price updates
   - All pools have `lastUpdatedAtBlock: 0` and never get updated
   - Edge widening depends on `lastUpdatedAtBlock === currentBlockNumber`, so edges never widen

2. **Temporary Solution**:
   - Initially added simulation code to randomly update 5-10 pool prices per block
   - Removed simulation at user's request, keeping only logging statements
   - WebSocket server needs to be fixed to send actual price updates with blocks

3. **Graph View Background Fix**:
   - Updated GraphViewContent.tsx background to match List View's warm cream theme
   - Changed from purple theme to `bg-[#FFF4E005]` with backdrop blur
   - Added gradient border effect for consistency

4. **Edge Tooltip Simplification**:
   - Removed pool link from edge tooltip - now displays pool ID as plain text
   - Removed redundant "Block" field since "Last Update" already shows temporal information
   - Edge tooltip now shows: Protocol, Token pair, Pool ID (with copy button), Fee, Last update, and Spot price

### Recent Development Updates (2025-05-26)

#### Critical Features to Maintain
1. **Filter Popover Infinite Scroll**: 
   - Token AND Pool ID popovers in PoolListFilterBar MUST have infinite scroll
   - Shows 100 items initially, loads 100 more when near bottom
   - Resets count to 100 when search changes
   - DO NOT remove this feature - it's essential for handling large lists
   - Batch size is 100 for both token and pool ID filters

2. **Pool ID Filter Enhancement**:
   - Shows pool ID with token pair: "0x001b...23d4 (TokenA / TokenB)"
   - Search works across pool ID AND token symbols
   - Full pool data is passed to filter bar (not just IDs)
   - Displays shortened pool ID with full token pair names

### Recent Development Updates (2025-05-26)

#### Swap Simulator Improvements
1. **Input Field Styling**: Fixed input amount capsule appearance
   - Replaced styled `Input` component with native `input` element
   - Removed all default padding, margins, and borders
   - Input and output amounts now have identical visual appearance
   - Both use `text-[28px]` for consistent sizing

2. **Pool ID Tooltip Interaction**: Fixed tooltip click behavior
   - Added `onClick={(e) => e.stopPropagation()}` to wrapper div
   - Prevents row selection when interacting with pool ID tooltip
   - Tooltip shows instantly with `delayDuration={0}`
   - Added `z-[100]` for proper layering

3. **Fee Percentage Formatting**: Added protocol-specific fee parsing
   - Added support for `ekubo_v2` protocol fee format (divides by 1e16)
   - Limited all fee percentages to 4 decimal places maximum
   - Example: ekubo fee `0x000346dc5d638865` now displays as "0.0922%" instead of "92233720368.5477%"
   - Consistent rounding applied across all protocols

#### UI Component Refinements
1. **Popover Glass Effect**: Updated all popovers to match Figma design
   - Background: `rgba(255, 244, 224, 0.04)` with `backdrop-blur-[104px]`
   - Shadow: `0px 4px 16px 0px rgba(37, 0, 63, 0.2)`
   - Border: `rgba(255, 255, 255, 0.1)`

2. **Checkbox Styling**: Changed from rounded to square checkboxes
   - Updated from `rounded-sm` to `rounded-none` for perfect square appearance
   - Border color: `rgba(255, 244, 224, 0.64)`

3. **Pool ID Tooltips**: Added hover tooltips for pool IDs
   - Shows full pool ID on hover for easy copying
   - Styled with dark background and backdrop blur
   - Uses `select-all` class for easy selection

4. **Swap Simulator Number Formatting**:
   - Input and output amounts use consistent `text-[28px]` sizing
   - Output amounts display maximum 2 decimal places
   - Smart decimal formatting: shows actual decimals up to 2 places (e.g., "273.42" not "273.420000")
   - Applied to both buy amount display and net amount

### Recent UI Color System Updates (2025-05-26)

The Pool List View UI has undergone significant color system updates to align with the TC Design from Figma:

1. **Color Scheme Transition**
   - Moved from cool purple/blue tones to warm cream/beige palette
   - Primary text color: `rgba(255, 244, 224, 1)` (#FFF4E0 at full opacity)
   - Background panels: `rgba(255, 244, 224, 0.02)` to `rgba(255, 244, 224, 0.05)`
   - Borders: `rgba(255, 244, 224, 0.05)` for table rows

2. **Component-Specific Changes**
   - **ListView.tsx**: Updated container with glass effect using backdrop blur and warm cream background
   - **PoolTable.tsx**: 
     - All text colors changed to `rgba(255, 244, 224, 1)` 
     - Selected rows use transparent warm backgrounds
     - Table row borders at 0.05 opacity
   - **PoolListFilterBar.tsx**: All popover text, inputs, and labels updated to warm white

3. **Design Patterns**
   - Gradient border implementation using nested divs with absolute positioning
   - Transparent overlays for interactive states
   - Consistent use of rgba values for precise color control

## Next Steps

1. Continue monitoring for any remaining color inconsistencies
2. Test the UI across different screen sizes and browsers
3. Implement any additional TC Design requirements as they arise

## Active Decisions and Considerations

### Color System
- **Decision**: Use `rgba(255, 244, 224, 1)` for ALL text with full opacity
- **Rationale**: User explicitly requested opacity 1 for maximum readability
- **Implementation**: Applied consistently across table cells, popovers, and all UI text

### Border Implementation
- **Decision**: Use solid borders with low opacity instead of gradient borders
- **Rationale**: Simpler implementation, better performance, cleaner appearance
- **Implementation**: `borderBottom: '1px solid rgba(255, 244, 224, 0.05)'`

### Glass Effect Pattern
- **Decision**: Use backdrop blur with semi-transparent backgrounds
- **Rationale**: Creates modern, layered UI as per TC Design
- **Implementation**: `backdrop-blur-[24px]` with `bg-[#FFF4E005]`

## Important Patterns and Preferences

### CSS Gradient Border Technique
When implementing gradient borders with border-radius:
1. Create wrapper div with gradient background and padding
2. Inner div with solid background color
3. This creates the appearance of a gradient border

Example:
```jsx
<div className="bg-gradient-to-b from-[color1] to-[color2] p-[1px] rounded-xl">
  <div className="bg-[solidColor] rounded-xl h-full w-full">
    {/* Content */}
  </div>
</div>
```

### Color Value Consistency
- Always use rgba format for colors to maintain consistency
- Full opacity (1) for primary text
- Lower opacity for secondary elements (0.6-0.8)
- Very low opacity for backgrounds (0.02-0.08)

### UI Component Guidelines
- **Token Address Display**: Always format raw addresses using `renderHexId` for consistency
- **Number Precision**: Never truncate amounts in Swap Simulator - show full precision
- **Popover Styling**: Use transparent borders with warm cream colors
- **Checkbox Styling**: Folly red (#FF3366) for checked state with white checkmark
- **Filter Token Sorting**: ALWAYS sort selected tokens first in filter popovers for consistency between views

## Learnings and Project Insights

1. **Figma Design Accuracy**: The TC Design uses a very specific warm color palette that must be matched exactly
2. **Opacity Matters**: Text readability is crucial - full opacity ensures maximum contrast
3. **Layered UI Design**: The app uses multiple transparent layers to create depth
4. **Performance Considerations**: Simple borders perform better than complex gradients for frequently rendered elements like table rows
5. **WebSocket Data Flow**: The WebSocket server sends initial pool data but doesn't include price updates with new blocks
6. **vis-network Edge Updates**: Edge widening requires careful management of DataSet updates and proper tracking of pool update blocks
7. **Debugging Complex Data Flows**: Color-coded console logging (🔷, 🔶, 🔸, etc.) helps trace data through multiple system layers