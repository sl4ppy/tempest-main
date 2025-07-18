# Tempest: Game Data and Content Reference

This section of the documentation provides a complete, reconstruction-grade reference for all of the game's data assets. The goal is to capture every value, table, and parameter required to replicate the game's content and behavior exactly as it was in the original.

This data is separated from the code documentation to provide a clear distinction between the game's logic and its content.

## Index of Data Files

*   **[Text and Language Data](./DATA_ASSETS.md#1-text-and-language-data)**
    *   Contains all user-facing strings for all four supported languages (English, German, French, Spanish). Includes high score tables, UI text, and in-game messages.

*   **[Vector Shape Data (`ALVROM.MAC`):](./DATA_ASSETS.md#2-vector-shapes)** See [`DATA_ASSETS.md`](./DATA_ASSETS.md#2-vector-shapes) for the full list of shapes.

*   **[Sound Effect Data](./DATA_ASSETS.md#3-sound-data)**
    *   The complete data tables used by the sound engine to synthesize every sound effect in the game.

*   **[Level and Wave Data](./LEVEL_DATA.md)**
    *   Defines the properties for each of the 16 unique levels, including well shape, colors, and enemy behavior.

*   **[Game Tuning and Constants](./GAME_TUNING.md)**
    *   A comprehensive list of constants and parameters that control the game's difficulty and feel, such as timings, speeds, and scoring.

*   **[Additional Data and Non-Tabular Content](./ADDITIONAL_DATA.md)**
    *   Describes how systems without discrete data tables (like [State Machines](./SYSTEMS.md#5-game-state-machine) and Particle Systems) are implemented, and clarifies the absence of rasterized assets like sprites. 