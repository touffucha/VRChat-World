# Udon Sharp Scripts Directory Structure

This directory contains all Udon Sharp scripts for the VRChat World project, organized by functionality.

## Directory Structure

### `/Player`
Player-related scripts including:
- Player movement and controls
- Player stats and attributes
- Player-specific interactions

### `/World`
World mechanics and systems:
- World state management
- Environment systems
- Global game mechanics

### `/UI`
User Interface scripts:
- Menu systems
- HUD elements
- UI interactions and animations

### `/Utility`
Helper scripts and common functions:
- Math utilities
- Data structures
- Common behaviors

### `/Networking`
Network-related functionality:
- Synchronization scripts
- Multiplayer features
- Network events

### `/Interaction`
Interactive object scripts:
- Pickups and items
- Buttons and switches
- Interactive world elements

### `/Audio`
Audio system scripts:
- Sound managers
- Music controllers
- Audio effects

### `/Debug`
Development and debugging tools:
- Debug visualizers
- Test scripts
- Performance monitoring

## Naming Convention

- Use PascalCase for script names (e.g., `PlayerController.cs`)
- Prefix with category when needed (e.g., `UI_MainMenu.cs`)
- Keep names descriptive and concise

## Best Practices

1. Place scripts in the most appropriate subdirectory
2. Create new subdirectories for specific features if needed
3. Keep related scripts together
4. Document complex scripts with comments
5. Follow VRChat and Udon Sharp best practices