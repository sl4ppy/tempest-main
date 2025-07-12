# Tempest: ALLANG.MAC - Language and Message Data

**File:** `ALLANG.MAC`
**Purpose:** This file is the central repository for all human-readable text strings displayed in the game. It contains a sophisticated, table-driven system for managing messages and their translations for four different languages: English, French, German, and Spanish.

---

## 1. Multi-Language Architecture

The core of this file is a set of parallel data tables that allow the game to select the correct language at runtime. This is a powerful feature for a game of this era.

The system is built around the following tables:
-   **Language Pointer Tables:** Four separate tables (`ENGMSG`, `FREMSG`, `GERMSG`, `SPAMSG`) each contain a list of pointers. Each entry in a table points to the raw vector data for a specific message in that language.
-   **Message Attribute Table (`MSGLBS`):** A single table that stores the properties for each message, such as its color, scale, and default Y-position on the screen. This table is shared across all languages.
-   **Literal Data:** The bulk of the file consists of the actual message strings, compiled into the byte-array format understood by the text renderer using the `.ASCVG` macro. There is a complete set of these strings for each of the four languages.

At runtime, the game sets a master pointer (`LITRAL`) to point to the start of the appropriate language table (e.g., `ENGMSG`). When the message engine needs to draw a message, it uses the message's ID to look up its properties from the shared attribute table and its text data from the language-specific pointer table.

## 2. The `MESS` Macro

The `MESS` macro is the organizational workhorse of this file. It is used to define a single message and all its associated properties and translations in one place.

A typical definition looks like this:
`MESS GAMOV,GREEN,1,56`

This single line does several things at once:
1.  **Creates a Message ID:** It defines a unique, global label for this message (`M'GAMOV`). This is the ID the rest of the game code uses to request this message.
2.  **Populates Pointer Tables:** It adds a pointer to the English "Game Over" string (`E'GAMOV`) to the `ENGMSG` table, a pointer to the French string (`F'GAMOV`) to the `FREMSG` table, and so on for German and Spanish.
3.  **Populates Attribute Table:** It adds the color (`GREEN`), scale (`1`), and Y-position (`56`) for this message into the shared `MSGLBS` attribute table.

This system keeps all the information for a single message neatly organized and makes it easy to add new messages or edit existing ones.

## 3. Subroutines

*(This file also contains game logic subroutines, such as `PRORAT` for the "Rate Yourself" screen. These will be documented after the rest of the file is analyzed.)*

## 3. `INILIT` (Initialize Literals)

This is a critical initialization subroutine called at the beginning of a new game. Its primary purpose is to read the physical DIP switch settings on the arcade cabinet's main board and configure the game's core parameters.

**Functionality:**
1.  **Read Bonus Life Setting:** It reads from DIP switch bank 1 (`INOP1`) to determine at what score interval the player receives a bonus life.
2.  **Read Coin Mode:** It reads from DIP switch bank 0 (`INOP0`) to determine the pricing structure (e.g., "Free Play", "1 Coin, 1 Play", "2 Coins, 1 Play") and stores this value in `$CMODE` for the `COIN65` library to use.
3.  **Read Lives per Game:** It reads from `INOP1` to set the number of lives the player starts with (2, 3, 4, or 5).
4.  **Set Language:** This is the most important function. It reads the language selection from the DIP switches. It then uses this value as an index into the `LNGTAB` table to find the starting address of the correct message pointer table (e.g., `ENGMSG` for English). It stores this address in the master `LITRAL` pointer. This single action ensures that all subsequent text displayed by the `MSGS` engine appears in the selected language.

---

This concludes the documentation for `ALLANG.MAC`. 