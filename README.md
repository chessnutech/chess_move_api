### Chessnut Move Chess Board API Documentation

#### **1. Bluetooth Connection**
- **Device Name**: "Chessnut Move"  
- **FEN Notification Service**:  
  - UUID: `1b7e8261-2877-41c3-b46e-cf057c562023`  
  - Characteristic (FEN data notification): `1b7e8262-2877-41c3-b46e-cf057c562023`  
- **Operation Commands Service**:  
  - UUID: `1b7e8271-2877-41c3-b46e-cf057c562023`  
  - Characteristic (Write commands): `1b7e8272-2877-41c3-b46e-cf057c562023`  
  - Characteristic (Response notifications): `1b7e8273-2877-41c3-b46e-cf057c562023`  

---

#### **2. Obtaining FEN (Board Position)**
After enabling notifications on the FEN characteristic, a **36-byte array** is received.  
- **Position Data**: Bytes `[2]` to `[33]` (32 bytes total) represent board state.  
- **Encoding**:  
  - Each byte encodes **two squares** (4 bits per square).  
  - **Square Order**: Starts at `h8` → `g8` → `f8` → ... → `a8` → `h7` → ... → `a1`.  
  - **Lower 4 bits** of a byte = First square (e.g., `h8`).  
  - **Higher 4 bits** = Next square (e.g., `g8`).  
- **Piece Mapping**:  
  ```javascript
  const pieces = ['', 'q', 'k', 'b', 'p', 'n', 'R', 'P', 'r', 'B', 'N', 'Q', 'K']; 
  // 0: Empty, 1: Black Queen, 2: Black King, ..., 12: White King.
  ```  
- **FEN Conversion (JavaScript Example)**:  
  ```javascript
  const pieces = ['', 'q', 'k', 'b', 'p', 'n', 'R', 'P', 'r', 'B', 'N', 'Q', 'K'];
  let fen = "";
  let empty = 0;
  for (let row = 0; row < 8; row++) {
    for (let col = 7; col >= 0; col--) {
      const index = Math.floor((row * 8 + col) / 2) + 2;
      const pieceVal = (col % 2 === 0) 
          ? data[index] & 0x0F 
          : data[index] >> 4;
      const piece = pieces[pieceVal];
      if (piece === '') empty++;
      else {
        if (empty > 0) fen += empty;
        fen += piece;
        empty = 0;
      }
    }
    if (empty > 0) fen += empty;
    if (row < 7) fen += '/';
    empty = 0;
  }
  ```

---

#### **3. Setting FEN for Auto-Moves**
Send a **35-byte command** to the Operation Write Characteristic:  
- **Command Structure**:  
  ```[0x42, 0x21, ...boardData..., forceFlag]```  
  - Bytes `[2]` to `[33]`: Board state (same encoding as FEN notification).  
  - Byte `[34]` (forceFlag):  
    - `0`: Force moves (override user interactions).  
    - `1`: Non-force (user-moved pieces halt auto-move).  
- **Stopping Auto-Moves**:  
  Send `[0x42, 0x21, ...34 zeros...]` (35 bytes).  
- **Note**: FEN notifications are **disabled** during auto-moves.  

---

#### **4. Setting LEDs**
Send a **34-byte command** to the Operation Write Characteristic:  
- **Command Structure**:  
  ```[0x43, 0x20, ...ledData...]```  
- **LED Encoding** (Bytes `[2]` to `[33]`):  
  - Each byte controls two squares (same order as FEN data).  
  - **Nibble Values**:  
    - `0`: LED off  
    - `1`: Red  
    - `2`: Green  
    - `3`: Blue  

---

#### **5. Reading Battery Level**
- **Command** (Write Characteristic):  
  `[0x41, 0x01, 0x0C]` (3 bytes).  
- **Response** (Notification Characteristic):  
  `[0x41, 0x03, 0x0C, charging, batteryLevel]`  
  - `charging`: `1` (charging), `0` (not charging).  
  - `batteryLevel`: `0`–`100` (percentage).  

---

#### **6. Reading Piece Status**
- **Command** (Write Characteristic):  
  `[0x41, 0x01, 0x0B]` (3 bytes).  
- **Response** (Notification Characteristic):  
  `[0x41, 0x89, 0x0B, ...pieceData...]`  
  - **Piece Order** (34 pieces):  
    ```javascript
    // White: Pawns×8, Rooks×2, Knights×2, Bishops×2, Queens×2, King×1
    // Black: Pawns×8, Rooks×2, Knights×2, Bishops×2, Queens×2, King×1
    const pieces = ['P','P','P','P','P','P','P','P','R','R','N','N','B','B','Q','Q','K',
                    'p','p','p','p','p','p','p','p','r','r','n','n','b','b','q','q','k'];
    ```  
  - **Per-Piece Data** (3 bytes per piece):  
    - `x`, `y`: Normalized coordinates (`0`–`255`).  
    - `bat`: Battery percentage (`0`–`100`).  

---
