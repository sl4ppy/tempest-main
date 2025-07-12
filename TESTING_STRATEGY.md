# TEMPEST TESTING STRATEGY
## Comprehensive Test Suite for Critical Components

### Table of Contents
1. [Testing Philosophy](#testing-philosophy)
2. [Test Categories](#test-categories)
3. [Critical Component Tests](#critical-component-tests)
4. [Integration Testing](#integration-testing)
5. [Performance Testing](#performance-testing)
6. [Verification & Validation](#verification--validation)
7. [Test Data and Fixtures](#test-data-and-fixtures)
8. [Continuous Integration](#continuous-integration)

---

## Testing Philosophy

### Core Principles
- **Accuracy First**: Every test must verify behavior against original hardware
- **Cycle-Level Precision**: Timing tests down to individual CPU cycles
- **Comprehensive Coverage**: 100% test coverage for critical paths
- **Regression Prevention**: Extensive test suite prevents functionality breaks
- **Performance Validation**: Maintain 60+ FPS under all conditions

### Testing Pyramid
```
    ┌─────────────┐
    │   E2E       │  ← Full game scenarios
    │   Tests     │
    ├─────────────┤
    │ Integration │  ← Component interaction
    │   Tests     │
    ├─────────────┤
    │   Unit      │  ← Individual functions
    │   Tests     │
    └─────────────┘
```

---

## Test Categories

### 1. Unit Tests
**Scope**: Individual functions and classes  
**Framework**: Google Test  
**Coverage Target**: 95%+

### 2. Integration Tests  
**Scope**: Component interactions  
**Framework**: Google Test + Custom harnesses  
**Coverage Target**: 90%+

### 3. System Tests
**Scope**: End-to-end game scenarios  
**Framework**: Behavior-driven testing  
**Coverage Target**: All game states

### 4. Performance Tests
**Scope**: Timing and optimization  
**Framework**: Google Benchmark  
**Coverage Target**: All critical paths

### 5. Regression Tests
**Scope**: Known working functionality  
**Framework**: Golden master testing  
**Coverage Target**: All major features

---

## Critical Component Tests

### 1. Mathbox Coprocessor Tests

#### 1.1 AMD 2901 ALU Tests
```cpp
class AMD2901Tests : public ::testing::Test {
protected:
    AMD2901 alu;
    
    void SetUp() override {
        alu.reset();
    }
};

// Test all 8 ALU functions
TEST_F(AMD2901Tests, ALUFunctions) {
    // Function 0: ADD
    alu.setA(0x1234);
    alu.setB(0x5678);
    auto result = alu.execute(ALU_FUNCTION_ADD, false);
    EXPECT_EQ(result.value, 0x68AC);
    EXPECT_FALSE(result.carry);
    
    // Function 1: SUBR (S - R)
    result = alu.execute(ALU_FUNCTION_SUBR, false);
    EXPECT_EQ(result.value, 0x4444);
    
    // Function 2: SUBS (R - S)  
    result = alu.execute(ALU_FUNCTION_SUBS, false);
    EXPECT_EQ(result.value, 0xBBBC);
    EXPECT_TRUE(result.carry);  // Borrow
    
    // Test all remaining functions...
}

// Test Q register operations
TEST_F(AMD2901Tests, QRegisterOperations) {
    alu.setQ(0xA5A5);
    
    // Test Q shift up (left)
    auto result = alu.execute_with_dest(DEST_RAMQU, false);
    EXPECT_EQ(alu.getQ(), 0x4B4A);  // Shifted left
    
    // Test Q shift down (right)
    alu.setQ(0xA5A5);
    result = alu.execute_with_dest(DEST_RAMQD, false);
    EXPECT_EQ(alu.getQ(), 0x52D2);  // Shifted right with carry
}

// Test register file operations
TEST_F(AMD2901Tests, RegisterFile) {
    // Test all 16 registers
    for (int i = 0; i < 16; i++) {
        uint16_t test_value = 0x1000 + i;
        alu.setRegister(i, test_value);
        EXPECT_EQ(alu.getRegister(i), test_value);
    }
    
    // Test register-to-register operations
    alu.setRegister(5, 0x1234);
    alu.setRegister(10, 0x5678);
    
    auto result = alu.execute_AB(5, 10, ALU_FUNCTION_ADD);
    EXPECT_EQ(result.value, 0x68AC);
}
```

#### 1.2 Microcode Execution Tests
```cpp
class MicrocodeTests : public ::testing::Test {
protected:
    MathboxCoprocessor mathbox;
    
public:
    // Test complete microcode programs
    TEST_F(MicrocodeTests, MultiplicationMicrocode) {
        // Load test values
        mathbox.setInputX(0x1234);
        mathbox.setInputY(0x5678);
        
        // Execute MULXP microcode sequence
        mathbox.startMicroprogram(MICROCODE_MULXP);
        
        // Verify cycle count
        int start_cycles = mathbox.getCycleCount();
        while (mathbox.isBusy()) {
            mathbox.step();
        }
        int total_cycles = mathbox.getCycleCount() - start_cycles;
        EXPECT_EQ(total_cycles, 46);  // Exact cycle count from original
        
        // Verify result
        uint32_t result = mathbox.getResult32();
        EXPECT_EQ(result, 0x1234UL * 0x5678UL);
    }
    
    TEST_F(MicrocodeTests, DivisionMicrocode) {
        // Test 32÷16 division
        mathbox.setDividend(0x12345678);
        mathbox.setDivisor(0x1234);
        
        mathbox.startMicroprogram(MICROCODE_DIVID);
        
        while (mathbox.isBusy()) {
            mathbox.step();
        }
        
        uint16_t quotient = mathbox.getQuotient();
        uint16_t remainder = mathbox.getRemainder();
        
        // Verify: dividend = quotient * divisor + remainder
        uint32_t check = (uint32_t)quotient * 0x1234 + remainder;
        EXPECT_EQ(check, 0x12345678);
    }
    
    TEST_F(MicrocodeTests, EdgeCases) {
        // Division by zero
        mathbox.setDividend(0x12345678);
        mathbox.setDivisor(0x0000);
        mathbox.startMicroprogram(MICROCODE_DIVID);
        
        while (mathbox.isBusy()) {
            mathbox.step();
        }
        
        // Should return maximum value or specific error code
        EXPECT_EQ(mathbox.getQuotient(), 0xFFFF);
        
        // Overflow cases
        mathbox.setInputX(0x8000);
        mathbox.setInputY(0x8000);
        mathbox.startMicroprogram(MICROCODE_MULXP);
        
        while (mathbox.isBusy()) {
            mathbox.step();
        }
        
        // 0x8000 * 0x8000 = 0x40000000
        EXPECT_EQ(mathbox.getResult32(), 0x40000000);
    }
};
```

#### 1.3 Mathematical Algorithm Tests
```cpp
class MathboxAlgorithmTests : public ::testing::Test {
public:
    TEST_F(MathboxAlgorithmTests, DistanceApproximation) {
        MathboxCoprocessor mathbox;
        
        // Test known distance calculations
        struct {
            int16_t dx, dy;
            uint16_t expected_distance;
        } test_cases[] = {
            {3, 4, 5},      // 3-4-5 triangle: should be ~5
            {5, 12, 13},    // 5-12-13 triangle: should be ~13
            {0, 10, 10},    // Vertical line: exact
            {10, 0, 10},    // Horizontal line: exact
            {7, 7, 9},      // 45-degree: sqrt(98) ≈ 9.9
        };
        
        for (const auto& test : test_cases) {
            mathbox.setDeltaX(test.dx);
            mathbox.setDeltaY(test.dy);
            mathbox.startMicroprogram(MICROCODE_DSTNCE);
            
            while (mathbox.isBusy()) {
                mathbox.step();
            }
            
            uint16_t result = mathbox.getDistance();
            
            // Allow small error due to approximation
            EXPECT_NEAR(result, test.expected_distance, 1);
        }
    }
    
    TEST_F(MathboxAlgorithmTests, ClippingAlgorithm) {
        MathboxCoprocessor mathbox;
        
        // Set clipping window bounds
        mathbox.setClipBounds(-100, 100, -100, 100);
        
        // Test point inside window
        mathbox.setPoint(50, 50);
        mathbox.startMicroprogram(MICROCODE_CLIP);
        while (mathbox.isBusy()) mathbox.step();
        EXPECT_TRUE(mathbox.isPointVisible());
        
        // Test point outside window
        mathbox.setPoint(150, 50);
        mathbox.startMicroprogram(MICROCODE_CLIP);
        while (mathbox.isBusy()) mathbox.step();
        EXPECT_FALSE(mathbox.isPointVisible());
        
        // Test clipped intersection point
        mathbox.setLineEndpoints(50, 50, 150, 150);
        mathbox.startMicroprogram(MICROCODE_CLIP);
        while (mathbox.isBusy()) mathbox.step();
        
        auto clipped_point = mathbox.getClippedEndpoint();
        EXPECT_EQ(clipped_point.x, 100);  // Should clip at window edge
        EXPECT_EQ(clipped_point.y, 100);
    }
};
```

### 2. Vector Graphics Engine Tests

#### 2.1 Instruction Decoding Tests
```cpp
class VectorInstructionTests : public ::testing::Test {
public:
    TEST_F(VectorInstructionTests, VCTRShortDecoding) {
        // Test short vector instruction: 0100 ZZZZ YYYY YXXX XX
        uint16_t instruction = 0x4000;  // Base opcode
        instruction |= (7 << 8);        // Intensity = 7
        instruction |= (10 << 3);       // Y = 10  
        instruction |= (5);             // X = 5
        
        auto decoded = VectorInstruction::decode(instruction);
        EXPECT_EQ(decoded->getType(), VectorInstruction::VCTR_SHORT);
        
        auto vctr = static_cast<VCTRShortInstruction*>(decoded.get());
        EXPECT_EQ(vctr->getDeltaX(), 5);
        EXPECT_EQ(vctr->getDeltaY(), 10);
        EXPECT_EQ(vctr->getIntensity(), 7);
    }
    
    TEST_F(VectorInstructionTests, VCTRLongDecoding) {
        // Test long vector instruction (32-bit)
        uint16_t word1 = 0x0123;  // Y component (13-bit signed)
        uint16_t word2 = 0x7456;  // X component + intensity
        
        auto decoded = VectorInstruction::decode(word1, word2);
        EXPECT_EQ(decoded->getType(), VectorInstruction::VCTR_LONG);
        
        auto vctr = static_cast<VCTRLongInstruction*>(decoded.get());
        
        // Extract Y (13-bit signed from word1)
        int16_t expected_y = (int16_t)(word1 << 3) >> 3;  // Sign extend
        EXPECT_EQ(vctr->getDeltaY(), expected_y);
        
        // Extract X (13-bit from word2) and intensity (3-bit)
        int16_t expected_x = (int16_t)(word2 << 3) >> 3;
        uint8_t expected_intensity = (word2 >> 13) & 0x7;
        EXPECT_EQ(vctr->getDeltaX(), expected_x);
        EXPECT_EQ(vctr->getIntensity(), expected_intensity);
    }
    
    TEST_F(VectorInstructionTests, ControlInstructions) {
        // HALT instruction
        auto halt = VectorInstruction::decode(0x2000);
        EXPECT_EQ(halt->getType(), VectorInstruction::HALT);
        
        // JSRL instruction
        auto jsrl = VectorInstruction::decode(0xA123);
        EXPECT_EQ(jsrl->getType(), VectorInstruction::JSRL);
        auto jsrl_instr = static_cast<JSRLInstruction*>(jsrl.get());
        EXPECT_EQ(jsrl_instr->getAddress(), 0x123 * 2);  // Address is word-based
        
        // SCAL instruction  
        auto scal = VectorInstruction::decode(0x7348);
        EXPECT_EQ(scal->getType(), VectorInstruction::SCAL);
        auto scal_instr = static_cast<SCALInstruction*>(scal.get());
        EXPECT_EQ(scal_instr->getBinaryScale(), 3);      // 3 bits
        EXPECT_EQ(scal_instr->getLinearScale(), 0x48);   // 8 bits
    }
};
```

#### 2.2 Display List Processing Tests
```cpp
class DisplayListTests : public ::testing::Test {
protected:
    VectorProcessor vg;
    MockMemoryBus memory;
    MockVectorDisplay display;
    
    void SetUp() override {
        vg.setMemoryBus(&memory);
        vg.setDisplay(&display);
    }
    
public:
    TEST_F(DisplayListTests, SimpleDisplayList) {
        // Create a simple display list
        uint16_t display_list[] = {
            0x8000,  // CNTR - center beam
            0x6000,  // STAT - set status  
            0x4123,  // VCTR short - draw vector
            0x4234,  // VCTR short - draw vector
            0x2000   // HALT - stop
        };
        
        // Load into memory
        uint16_t start_addr = 0x2000;
        for (size_t i = 0; i < sizeof(display_list)/sizeof(uint16_t); i++) {
            memory.writeWord(start_addr + i*2, display_list[i]);
        }
        
        // Execute display list
        vg.startExecution(start_addr);
        
        int instruction_count = 0;
        while (vg.isExecuting()) {
            vg.step();
            instruction_count++;
            
            // Prevent infinite loop
            ASSERT_LT(instruction_count, 100);
        }
        
        // Verify execution
        EXPECT_EQ(instruction_count, 5);  // 5 instructions
        EXPECT_TRUE(vg.isHalted());
        
        // Verify rendering calls
        EXPECT_EQ(display.getCenterCallCount(), 1);
        EXPECT_EQ(display.getVectorCallCount(), 2);
    }
    
    TEST_F(DisplayListTests, SubroutineExecution) {
        // Main display list with subroutine call
        uint16_t main_list[] = {
            0xA100,  // JSRL $200 (subroutine at 0x200)
            0x2000   // HALT
        };
        
        // Subroutine 
        uint16_t subroutine[] = {
            0x4111,  // VCTR short
            0x4222,  // VCTR short  
            0xC000   // RTSL - return
        };
        
        // Load main list at 0x2000
        memory.writeWord(0x2000, main_list[0]);
        memory.writeWord(0x2002, main_list[1]);
        
        // Load subroutine at 0x200 (address from JSRL * 2)
        memory.writeWord(0x0200, subroutine[0]);
        memory.writeWord(0x0202, subroutine[1]);
        memory.writeWord(0x0204, subroutine[2]);
        
        vg.startExecution(0x2000);
        
        while (vg.isExecuting()) {
            vg.step();
        }
        
        EXPECT_TRUE(vg.isHalted());
        EXPECT_EQ(display.getVectorCallCount(), 2);  // 2 vectors in subroutine
    }
    
    TEST_F(DisplayListTests, NestedSubroutines) {
        // Test 5-level subroutine stack depth
        // Main -> Sub1 -> Sub2 -> Sub3 -> Sub4 -> Sub5
        
        // Each subroutine calls the next, last one returns
        // Verify stack operations work correctly
        
        // Create nested call structure
        memory.writeWord(0x2000, 0xA100);  // JSRL Sub1
        memory.writeWord(0x2002, 0x2000);  // HALT
        
        memory.writeWord(0x0200, 0xA200);  // Sub1: JSRL Sub2  
        memory.writeWord(0x0202, 0xC000);  // RTSL
        
        memory.writeWord(0x0400, 0xA300);  // Sub2: JSRL Sub3
        memory.writeWord(0x0402, 0xC000);  // RTSL
        
        memory.writeWord(0x0600, 0xA400);  // Sub3: JSRL Sub4
        memory.writeWord(0x0602, 0xC000);  // RTSL
        
        memory.writeWord(0x0800, 0xA500);  // Sub4: JSRL Sub5
        memory.writeWord(0x0802, 0xC000);  // RTSL
        
        memory.writeWord(0x0A00, 0x4123);  // Sub5: VCTR
        memory.writeWord(0x0A02, 0xC000);  // RTSL
        
        vg.startExecution(0x2000);
        
        while (vg.isExecuting()) {
            vg.step();
        }
        
        EXPECT_TRUE(vg.isHalted());
        EXPECT_EQ(vg.getStackDepth(), 0);  // Stack should be empty
        EXPECT_EQ(display.getVectorCallCount(), 1);
    }
};
```

#### 2.3 Vector Rendering Tests
```cpp
class VectorRenderingTests : public ::testing::Test {
protected:
    VectorDisplay display;
    
public:
    TEST_F(VectorRenderingTests, BasicVectorDrawing) {
        display.beginFrame();
        
        // Draw test vectors
        display.drawVector(0, 0, 100, 100, 7, COLOR_WHITE);    // Diagonal
        display.drawVector(0, 100, 100, 0, 5, COLOR_GREEN);    // Other diagonal
        display.drawVector(-50, 0, 50, 0, 3, COLOR_RED);       // Horizontal
        
        display.endFrame();
        
        // Verify vertex buffer contains correct data
        auto vertices = display.getFrameVertices();
        EXPECT_EQ(vertices.size(), 6);  // 3 lines * 2 vertices each
        
        // Check first line
        EXPECT_EQ(vertices[0].x, 0);
        EXPECT_EQ(vertices[0].y, 0);
        EXPECT_EQ(vertices[1].x, 100);
        EXPECT_EQ(vertices[1].y, 100);
        EXPECT_EQ(vertices[0].intensity, 7);
        EXPECT_EQ(vertices[0].color, COLOR_WHITE);
    }
    
    TEST_F(VectorRenderingTests, ScalingOperations) {
        display.beginFrame();
        
        // Set binary scale (1/4 size)
        display.setBinaryScale(2);  // 2^2 = 4x reduction
        
        // Draw vector - should be scaled
        display.drawVector(0, 0, 400, 400, 7, COLOR_WHITE);
        
        display.endFrame();
        
        auto vertices = display.getFrameVertices();
        
        // Vector should be scaled to 1/4 size
        EXPECT_EQ(vertices[1].x, 100);  // 400 / 4
        EXPECT_EQ(vertices[1].y, 100);  // 400 / 4
    }
    
    TEST_F(VectorRenderingTests, IntensityMapping) {
        display.beginFrame();
        
        // Test all intensity levels (0-7)
        for (int intensity = 0; intensity <= 7; intensity++) {
            display.drawVector(intensity * 10, 0, intensity * 10, 100, 
                             intensity, COLOR_WHITE);
        }
        
        display.endFrame();
        
        auto vertices = display.getFrameVertices();
        
        // Verify intensity mapping
        for (int i = 0; i < 8; i++) {
            float expected_alpha = i / 7.0f;  // Normalize to 0.0-1.0
            EXPECT_NEAR(vertices[i*2].intensity, expected_alpha, 0.01f);
        }
    }
};
```

### 3. CPU Emulation Tests

#### 3.1 Instruction Set Tests
```cpp
class CPU6502InstructionTests : public ::testing::Test {
protected:
    CPU6502 cpu;
    MockMemoryBus memory;
    
    void SetUp() override {
        cpu.setMemoryBus(&memory);
        cpu.reset();
    }
    
public:
    TEST_F(CPU6502InstructionTests, LoadStoreInstructions) {
        // LDA Immediate
        memory.write(0x0000, 0xA9);  // LDA #$42
        memory.write(0x0001, 0x42);
        
        int cycles = cpu.step();
        EXPECT_EQ(cycles, 2);
        EXPECT_EQ(cpu.getA(), 0x42);
        EXPECT_FALSE(cpu.getFlag(CPU6502::FLAG_ZERO));
        EXPECT_FALSE(cpu.getFlag(CPU6502::FLAG_NEGATIVE));
        
        // STA Zero Page
        memory.write(0x0002, 0x85);  // STA $10
        memory.write(0x0003, 0x10);
        
        cycles = cpu.step();
        EXPECT_EQ(cycles, 3);
        EXPECT_EQ(memory.read(0x10), 0x42);
    }
    
    TEST_F(CPU6502InstructionTests, ArithmeticInstructions) {
        // Set up for ADC
        cpu.setA(0x40);
        cpu.setFlag(CPU6502::FLAG_CARRY, false);
        
        memory.write(0x0000, 0x69);  // ADC #$30
        memory.write(0x0001, 0x30);
        
        int cycles = cpu.step();
        EXPECT_EQ(cycles, 2);
        EXPECT_EQ(cpu.getA(), 0x70);
        EXPECT_FALSE(cpu.getFlag(CPU6502::FLAG_CARRY));
        EXPECT_FALSE(cpu.getFlag(CPU6502::FLAG_ZERO));
        EXPECT_FALSE(cpu.getFlag(CPU6502::FLAG_NEGATIVE));
        
        // Test overflow
        cpu.setA(0x7F);
        memory.write(0x0002, 0x69);  // ADC #$01  
        memory.write(0x0003, 0x01);
        
        cycles = cpu.step();
        EXPECT_EQ(cpu.getA(), 0x80);
        EXPECT_TRUE(cpu.getFlag(CPU6502::FLAG_OVERFLOW));
        EXPECT_TRUE(cpu.getFlag(CPU6502::FLAG_NEGATIVE));
    }
    
    TEST_F(CPU6502InstructionTests, BranchInstructions) {
        // BEQ (branch on zero flag set)
        cpu.setFlag(CPU6502::FLAG_ZERO, true);
        cpu.setPC(0x1000);
        
        memory.write(0x1000, 0xF0);  // BEQ +$10
        memory.write(0x1001, 0x10);
        
        int cycles = cpu.step();
        EXPECT_EQ(cycles, 3);  // 2 base + 1 for branch taken
        EXPECT_EQ(cpu.getPC(), 0x1012);  // 0x1002 + 0x10
        
        // Test branch not taken
        cpu.setFlag(CPU6502::FLAG_ZERO, false);
        cpu.setPC(0x1020);
        
        memory.write(0x1020, 0xF0);  // BEQ +$10
        memory.write(0x1021, 0x10);
        
        cycles = cpu.step();
        EXPECT_EQ(cycles, 2);  // 2 base cycles
        EXPECT_EQ(cpu.getPC(), 0x1022);  // No branch
    }
    
    TEST_F(CPU6502InstructionTests, InterruptHandling) {
        // Set up interrupt vectors
        memory.write(0xFFFE, 0x00);  // IRQ vector low
        memory.write(0xFFFF, 0x80);  // IRQ vector high
        
        cpu.setPC(0x1234);
        cpu.setA(0x42);
        cpu.setFlag(CPU6502::FLAG_INTERRUPT, false);
        
        // Trigger IRQ
        cpu.triggerIRQ();
        
        // IRQ should be processed on next instruction
        memory.write(0x1234, 0xEA);  // NOP
        int cycles = cpu.step();
        
        // Verify IRQ handling
        EXPECT_EQ(cpu.getPC(), 0x8000);  // Jumped to IRQ vector
        EXPECT_TRUE(cpu.getFlag(CPU6502::FLAG_INTERRUPT));  // I flag set
        
        // Verify stack
        uint16_t saved_pc_low = memory.read(0x0100 + cpu.getSP() + 1);
        uint16_t saved_pc_high = memory.read(0x0100 + cpu.getSP() + 2);
        uint16_t saved_pc = saved_pc_low | (saved_pc_high << 8);
        EXPECT_EQ(saved_pc, 0x1235);  // PC + 1 saved on stack
        
        uint8_t saved_status = memory.read(0x0100 + cpu.getSP() + 3);
        // Status should have break flag clear (IRQ vs BRK)
        EXPECT_FALSE(saved_status & CPU6502::FLAG_BREAK);
    }
};
```

#### 3.2 Timing Accuracy Tests
```cpp
class CPU6502TimingTests : public ::testing::Test {
public:
    TEST_F(CPU6502TimingTests, InstructionCycles) {
        CPU6502 cpu;
        MockMemoryBus memory;
        cpu.setMemoryBus(&memory);
        cpu.reset();
        
        struct TimingTest {
            std::vector<uint8_t> instruction;
            int expected_cycles;
            const char* description;
        };
        
        std::vector<TimingTest> tests = {
            {{0xEA}, 2, "NOP"},
            {{0xA9, 0x42}, 2, "LDA Immediate"},
            {{0xAD, 0x00, 0x30}, 4, "LDA Absolute"},
            {{0xBD, 0x00, 0x30}, 4, "LDA Absolute,X (no page cross)"},
            {{0xBD, 0xFF, 0x30}, 5, "LDA Absolute,X (page cross)"},
            {{0x85, 0x10}, 3, "STA Zero Page"},
            {{0x8D, 0x00, 0x30}, 4, "STA Absolute"},
            {{0x20, 0x00, 0x80}, 6, "JSR"},
            {{0x60}, 6, "RTS"},
            {{0x4C, 0x00, 0x80}, 3, "JMP Absolute"},
            {{0x6C, 0x00, 0x80}, 5, "JMP Indirect"},
        };
        
        for (const auto& test : tests) {
            cpu.reset();
            
            // Load instruction
            for (size_t i = 0; i < test.instruction.size(); i++) {
                memory.write(0x8000 + i, test.instruction[i]);
            }
            
            cpu.setPC(0x8000);
            
            int cycles = cpu.step();
            EXPECT_EQ(cycles, test.expected_cycles) 
                << "Instruction: " << test.description;
        }
    }
    
    TEST_F(CPU6502TimingTests, PageCrossPenalties) {
        CPU6502 cpu;
        MockMemoryBus memory;
        cpu.setMemoryBus(&memory);
        
        // Test LDA Absolute,X with page boundary cross
        cpu.reset();
        cpu.setX(0x10);  // X register = 0x10
        
        // LDA $30FF,X - will cross page boundary (30FF + 10 = 310F)
        memory.write(0x8000, 0xBD);
        memory.write(0x8001, 0xFF);
        memory.write(0x8002, 0x30);
        
        cpu.setPC(0x8000);
        int cycles = cpu.step();
        EXPECT_EQ(cycles, 5);  // 4 base + 1 page cross penalty
        
        // Test without page boundary cross
        cpu.reset();
        cpu.setX(0x01);  // X register = 0x01
        
        // LDA $3000,X - no page cross (3000 + 01 = 3001)
        memory.write(0x8000, 0xBD);
        memory.write(0x8001, 0x00);
        memory.write(0x8002, 0x30);
        
        cpu.setPC(0x8000);
        cycles = cpu.step();
        EXPECT_EQ(cycles, 4);  // 4 base cycles only
    }
};
```

### 4. Game Logic Tests

#### 4.1 Collision Detection Tests
```cpp
class CollisionTests : public ::testing::Test {
protected:
    CollisionSystem collision;
    
public:
    TEST_F(CollisionTests, PlayerEnemyCollision) {
        // Create player at line 8
        Player player;
        player.setPosition(8, 0x10);  // Line 8, depth 0x10
        
        // Create enemy at same line and depth
        FlipperEntity enemy;
        enemy.setPosition(8, 0x10);
        
        EXPECT_TRUE(collision.checkPlayerEnemyCollision(player, enemy));
        
        // Move enemy to different line
        enemy.setPosition(9, 0x10);
        EXPECT_FALSE(collision.checkPlayerEnemyCollision(player, enemy));
        
        // Move enemy to same line but different depth
        enemy.setPosition(8, 0x20);
        EXPECT_FALSE(collision.checkPlayerEnemyCollision(player, enemy));
    }
    
    TEST_F(CollisionTests, ProjectileEnemyCollision) {
        PlayerBullet bullet;
        bullet.setPosition(5, 0x30);
        bullet.setVelocity(0, 9);  // Moving up at 9 units/frame
        
        FlipperEntity enemy;
        enemy.setPosition(5, 0x48);  // Same line, deeper
        
        // Bullet should hit enemy in 2 frames: 0x30 + 9*2 = 0x42, close to 0x48
        bool hit = false;
        for (int frame = 0; frame < 5; frame++) {
            bullet.update();
            if (collision.checkProjectileEnemyCollision(bullet, enemy)) {
                hit = true;
                break;
            }
        }
        
        EXPECT_TRUE(hit);
        
        // Test bullet can hit multiple enemies
        FlipperEntity enemy2;
        enemy2.setPosition(5, 0x60);
        
        bullet.resetHitCount();
        
        // First hit
        EXPECT_TRUE(collision.checkProjectileEnemyCollision(bullet, enemy));
        collision.processHit(bullet, enemy);
        EXPECT_EQ(bullet.getHitCount(), 1);
        EXPECT_TRUE(bullet.isActive());  // Still active after 1 hit
        
        // Second hit
        bullet.setPosition(5, 0x60);
        EXPECT_TRUE(collision.checkProjectileEnemyCollision(bullet, enemy2));
        collision.processHit(bullet, enemy2);
        EXPECT_EQ(bullet.getHitCount(), 2);
        EXPECT_FALSE(bullet.isActive());  // Deactivated after 2 hits
    }
    
    TEST_F(CollisionTests, SpikeCollision) {
        SpikeSystem spikes;
        Player player;
        
        // Create spike on line 3 at depth 0x40
        SpikerEntity spiker;
        spiker.setPosition(3, 0x50);
        spiker.setActive(true);
        
        spikes.updateSpikes(spiker);
        
        // Player drops through line 3
        player.setPosition(3, 0x10);  // At rim
        player.setState(Player::DROPPING);
        
        // Simulate drop
        for (int frame = 0; frame < 10; frame++) {
            player.updateDrop();
            
            if (spikes.checkSpikeCollision(player)) {
                EXPECT_GT(player.getDepth(), 0x40);  // Should hit at spike depth
                break;
            }
        }
        
        EXPECT_EQ(player.getState(), Player::DEAD);
    }
};
```

#### 4.2 Game State Tests
```cpp
class GameStateTests : public ::testing::Test {
protected:
    GameStateMachine game_state;
    MockInputManager input;
    
public:
    TEST_F(GameStateTests, AttractModeSequence) {
        game_state.setState(GameState::ATTRACT_DELAY);
        
        // Verify attract mode progression
        std::vector<GameState> expected_sequence = {
            GameState::ATTRACT_DELAY,
            GameState::ATTRACT_PAUSE,
            GameState::ATTRACT_LOGO,
            GameState::ATTRACT_PAUSE,
            GameState::ATTRACT_INIT,
            GameState::SKILL_SELECT,
            GameState::GAMEPLAY  // Demo play
        };
        
        for (GameState expected : expected_sequence) {
            EXPECT_EQ(game_state.getCurrentState(), expected);
            
            // Run state until transition
            while (game_state.getCurrentState() == expected) {
                game_state.update();
            }
        }
        
        // Should loop back to attract after demo
        // ... (continue testing loop)
    }
    
    TEST_F(GameStateTests, GameplayTransitions) {
        game_state.setState(GameState::GAMEPLAY);
        
        MockPlayer player;
        player.setLives(3);
        
        // Simulate player death
        player.kill();
        game_state.notifyPlayerDeath();
        
        EXPECT_EQ(game_state.getCurrentState(), GameState::END_LIFE);
        
        // Should transition to new life if lives remain
        game_state.update();
        EXPECT_EQ(game_state.getCurrentState(), GameState::NEW_LIFE);
        
        // Continue to gameplay
        game_state.update();
        EXPECT_EQ(game_state.getCurrentState(), GameState::GAMEPLAY);
        
        // Kill player with no lives remaining
        player.setLives(0);
        player.kill();
        game_state.notifyPlayerDeath();
        
        game_state.update();
        EXPECT_EQ(game_state.getCurrentState(), GameState::END_GAME);
    }
    
    TEST_F(GameStateTests, WaveProgression) {
        game_state.setState(GameState::GAMEPLAY);
        
        MockEnemyManager enemies;
        enemies.clearAllEnemies();  // Simulate wave complete
        
        game_state.notifyWaveComplete();
        EXPECT_EQ(game_state.getCurrentState(), GameState::END_WAVE);
        
        // Should show score bonus
        game_state.update();
        EXPECT_EQ(game_state.getCurrentState(), GameState::NEW_WAVE);
        
        // Should return to gameplay with next wave
        game_state.update();
        EXPECT_EQ(game_state.getCurrentState(), GameState::GAMEPLAY);
        EXPECT_EQ(game_state.getCurrentWave(), 2);
    }
};
```

#### 4.3 Scoring System Tests
```cpp
class ScoringTests : public ::testing::Test {
public:
    TEST_F(ScoringTests, EnemyPointValues) {
        ScoreManager score;
        
        // Test enemy point values from TUPSCM table
        EXPECT_EQ(score.getEnemyPoints(FLIPPER), 50);
        EXPECT_EQ(score.getEnemyPoints(TANKER), 100);
        EXPECT_EQ(score.getEnemyPoints(SPIKER), 200);
        EXPECT_EQ(score.getEnemyPoints(FUSEBALL), 750);
        EXPECT_EQ(score.getEnemyPoints(PULSAR), 200);
    }
    
    TEST_F(ScoringTests, WaveBonusCalculation) {
        ScoreManager score;
        
        // Wave bonus = base points * wave number
        for (int wave = 1; wave <= 16; wave++) {
            int expected_bonus = wave * 1000;  // Base bonus = 1000
            EXPECT_EQ(score.calculateWaveBonus(wave), expected_bonus);
        }
        
        // Test bonus caps at some point
        EXPECT_LE(score.calculateWaveBonus(99), 16000);
    }
    
    TEST_F(ScoringTests, ExtraLifeAwards) {
        ScoreManager score;
        LifeManager lives;
        
        // Extra life at 20,000 points
        score.addPoints(19999);
        EXPECT_EQ(lives.getLives(), 4);  // Starting lives
        
        score.addPoints(1);  // Reach 20,000
        EXPECT_EQ(lives.getLives(), 5);  // Extra life awarded
        
        // Extra life at 60,000 points
        score.addPoints(39999);
        EXPECT_EQ(lives.getLives(), 5);  // No change
        
        score.addPoints(1);  // Reach 60,000
        EXPECT_EQ(lives.getLives(), 6);  // Another extra life
        
        // No more extra lives after 60,000
        score.addPoints(100000);
        EXPECT_EQ(lives.getLives(), 6);  // No more awards
    }
};
```

---

## Integration Testing

### 1. Hardware Component Integration
```cpp
class HardwareIntegrationTests : public ::testing::Test {
protected:
    CPU6502 cpu;
    MathboxCoprocessor mathbox;
    VectorProcessor vector_graphics;
    MemoryBus memory_bus;
    
    void SetUp() override {
        // Connect all components
        cpu.setMemoryBus(&memory_bus);
        mathbox.setMemoryBus(&memory_bus);
        vector_graphics.setMemoryBus(&memory_bus);
        
        // Map hardware registers
        memory_bus.mapDevice(0xD100, 0xD11F, &mathbox);
        memory_bus.mapDevice(0xD500, 0xD501, &vector_graphics);
    }
    
public:
    TEST_F(HardwareIntegrationTests, MathboxCPUCommunication) {
        // CPU writes to Mathbox registers
        cpu.reset();
        
        // Load simple program: write to Mathbox, read result
        std::vector<uint8_t> program = {
            0xA9, 0x34,        // LDA #$34
            0x8D, 0x01, 0xD1,  // STA $D101 (RXL)
            0xA9, 0x12,        // LDA #$12  
            0x8D, 0x02, 0xD1,  // STA $D102 (RXH)
            0xA9, 0x78,        // LDA #$78
            0x8D, 0x03, 0xD1,  // STA $D103 (RYL)
            0xA9, 0x56,        // LDA #$56
            0x8D, 0x0B, 0xD1,  // STA $D10B (YHSP - start operation)
            // Wait loop
            0xAD, 0x00, 0xD1,  // LDA $D100 (status)
            0x30, 0xFB,        // BMI -5 (wait while busy)
            // Read result
            0xAD, 0x04, 0xD1,  // LDA result low
            0xAD, 0x05, 0xD1,  // LDA result high
        };
        
        // Load program into memory
        for (size_t i = 0; i < program.size(); i++) {
            memory_bus.write(0x8000 + i, program[i]);
        }
        
        cpu.setPC(0x8000);
        
        // Execute until Mathbox operation starts
        while (cpu.getPC() < 0x8010) {
            cpu.step();
        }
        
        // Verify Mathbox received values
        EXPECT_EQ(mathbox.getRegister(MathboxRegister::RXL), 0x34);
        EXPECT_EQ(mathbox.getRegister(MathboxRegister::RXH), 0x12);
        EXPECT_EQ(mathbox.getRegister(MathboxRegister::RYL), 0x78);
        EXPECT_EQ(mathbox.getRegister(MathboxRegister::RYH), 0x56);
        
        // Mathbox should be busy
        EXPECT_TRUE(mathbox.isBusy());
        
        // Run system until Mathbox completes
        while (mathbox.isBusy()) {
            mathbox.step();
            cpu.step();
        }
        
        // CPU should have read result
        uint32_t expected_result = 0x1234 * 0x5678;
        EXPECT_EQ(cpu.getA(), expected_result & 0xFF);  // Low byte in A
    }
    
    TEST_F(HardwareIntegrationTests, VectorGraphicsSynchronization) {
        // Test VG start/stop and HALT flag integration
        
        // CPU program to start VG
        std::vector<uint8_t> program = {
            0xA9, 0x00,        // LDA #$00
            0x8D, 0x01, 0xD5,  // STA $D501 (VGSTART)
            // Wait for VG completion
            0xAD, 0x10, 0xD0,  // LDA $D010 (IN1 register)
            0x10, 0xFB,        // BPL -5 (wait for HALT flag)
        };
        
        // Create simple display list
        std::vector<uint16_t> display_list = {
            0x8000,  // CNTR
            0x4123,  // VCTR
            0x2000   // HALT
        };
        
        // Load display list into Vector RAM
        for (size_t i = 0; i < display_list.size(); i++) {
            memory_bus.writeWord(0x2000 + i*2, display_list[i]);
        }
        
        // Load CPU program
        for (size_t i = 0; i < program.size(); i++) {
            memory_bus.write(0x8000 + i, program[i]);
        }
        
        cpu.setPC(0x8000);
        vector_graphics.setDisplayListStart(0x2000);
        
        // Execute CPU program
        while (cpu.getPC() < 0x8006) {
            cpu.step();
        }
        
        // VG should be executing
        EXPECT_TRUE(vector_graphics.isExecuting());
        
        // Run both systems
        while (vector_graphics.isExecuting()) {
            vector_graphics.step();
            cpu.step();
        }
        
        // CPU should have detected HALT
        EXPECT_TRUE(memory_bus.read(0xD010) & 0x40);  // HALT flag set
        
        // CPU should have completed wait loop
        EXPECT_EQ(cpu.getPC(), 0x800A);
    }
};
```

### 2. Full Game Scenario Tests
```cpp
class GameScenarioTests : public ::testing::Test {
protected:
    TempestGame game;
    
public:
    TEST_F(GameScenarioTests, CompleteAttractMode) {
        game.initialize();
        game.setState(GameState::ATTRACT_DELAY);
        
        // Run complete attract mode sequence
        auto start_time = std::chrono::high_resolution_clock::now();
        
        while (game.getState() != GameState::ATTRACT_DELAY || 
               game.getFrameCount() < 2) {  // Complete one full cycle
            game.updateFrame();
            
            // Prevent infinite loops
            auto current_time = std::chrono::high_resolution_clock::now();
            auto elapsed = std::chrono::duration_cast<std::chrono::seconds>(
                current_time - start_time).count();
            ASSERT_LT(elapsed, 60);  // Max 60 seconds
        }
        
        // Verify attract mode completed properly
        EXPECT_GT(game.getFrameCount(), 1000);  // Reasonable frame count
    }
    
    TEST_F(GameScenarioTests, SingleWavePlaythrough) {
        game.initialize();
        game.startGame(1);  // Start at wave 1
        
        MockInputManager input;
        game.setInputManager(&input);
        
        int initial_score = game.getScore();
        int initial_lives = game.getLives();
        
        // Simulate destroying all enemies
        auto enemies = game.getEnemyManager().getAllEnemies();
        for (auto& enemy : enemies) {
            // Position player to hit enemy
            input.setRotaryPosition(enemy->getLineNumber() * 16);
            
            // Fire at enemy
            input.setFirePressed(true);
            game.updateFrame();
            input.setFirePressed(false);
            
            // Update until bullet hits
            for (int frame = 0; frame < 100; frame++) {
                game.updateFrame();
                if (!enemy->isActive()) break;
            }
            
            EXPECT_FALSE(enemy->isActive());
        }
        
        // Wait for wave end
        while (game.getState() != GameState::END_WAVE) {
            game.updateFrame();
        }
        
        // Verify scoring
        EXPECT_GT(game.getScore(), initial_score);
        EXPECT_EQ(game.getLives(), initial_lives);  // No deaths
        EXPECT_EQ(game.getCurrentWave(), 2);  // Advanced to wave 2
    }
    
    TEST_F(GameScenarioTests, SuperzapperFunctionality) {
        game.initialize();
        game.startGame(1);
        
        MockInputManager input;
        game.setInputManager(&input);
        
        int initial_zappers = game.getSuperzapperCount();
        int initial_score = game.getScore();
        
        // Activate superzapper
        input.setSuperzapperPressed(true);
        game.updateFrame();
        input.setSuperzapperPressed(false);
        
        EXPECT_EQ(game.getState(), GameState::SUPERZAPPER);
        EXPECT_EQ(game.getSuperzapperCount(), initial_zappers - 1);
        
        // Run superzapper sequence
        while (game.getState() == GameState::SUPERZAPPER) {
            game.updateFrame();
        }
        
        // All enemies should be destroyed
        auto enemies = game.getEnemyManager().getAllEnemies();
        for (const auto& enemy : enemies) {
            EXPECT_FALSE(enemy->isActive());
        }
        
        // Score should increase
        EXPECT_GT(game.getScore(), initial_score);
        
        // Should return to gameplay
        EXPECT_EQ(game.getState(), GameState::GAMEPLAY);
    }
};
```

---

## Performance Testing

### 1. Frame Rate Verification
```cpp
class PerformanceTests : public ::testing::Test {
public:
    TEST_F(PerformanceTests, SustainedFrameRate) {
        TempestGame game;
        game.initialize();
        game.startGame(16);  // High wave with many enemies
        
        const int TEST_FRAMES = 3600;  // 1 minute at 60 FPS
        
        auto start_time = std::chrono::high_resolution_clock::now();
        
        for (int frame = 0; frame < TEST_FRAMES; frame++) {
            game.updateFrame();
        }
        
        auto end_time = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(
            end_time - start_time);
        
        // Should complete in approximately 60 seconds (60 FPS)
        double actual_fps = (TEST_FRAMES * 1000.0) / duration.count();
        
        EXPECT_GE(actual_fps, 59.0);  // Allow small variance
        EXPECT_LE(actual_fps, 61.0);
    }
    
    TEST_F(PerformanceTests, WorstCaseScenario) {
        TempestGame game;
        game.initialize();
        
        // Create worst-case scenario: maximum entities
        auto& enemy_manager = game.getEnemyManager();
        auto& bullet_manager = game.getBulletManager();
        
        // Spawn maximum enemies
        for (int line = 0; line < 16; line++) {
            for (int depth = 0x20; depth < 0xE0; depth += 0x10) {
                if (enemy_manager.getEnemyCount() < MAX_ENEMIES) {
                    enemy_manager.spawnFlipperAt(line, depth);
                }
            }
        }
        
        // Spawn maximum bullets
        for (int i = 0; i < MAX_PLAYER_BULLETS; i++) {
            bullet_manager.firePlayerBullet(i, 0x10);
        }
        
        for (int i = 0; i < MAX_ENEMY_BULLETS; i++) {
            bullet_manager.fireEnemyBullet(i % 16, 0x80);
        }
        
        // Measure frame time under stress
        const int STRESS_FRAMES = 300;  // 5 seconds
        std::vector<double> frame_times;
        
        for (int frame = 0; frame < STRESS_FRAMES; frame++) {
            auto frame_start = std::chrono::high_resolution_clock::now();
            
            game.updateFrame();
            
            auto frame_end = std::chrono::high_resolution_clock::now();
            auto frame_time = std::chrono::duration_cast<std::chrono::microseconds>(
                frame_end - frame_start).count() / 1000.0;
            
            frame_times.push_back(frame_time);
        }
        
        // Calculate statistics
        double avg_frame_time = std::accumulate(frame_times.begin(), 
                                               frame_times.end(), 0.0) / frame_times.size();
        double max_frame_time = *std::max_element(frame_times.begin(), frame_times.end());
        
        // Frame time should be well under 16.67ms (60 FPS)
        EXPECT_LT(avg_frame_time, 10.0);  // Average under 10ms
        EXPECT_LT(max_frame_time, 16.0);  // Worst case under 16ms
    }
};
```

### 2. Memory Usage Tests
```cpp
class MemoryTests : public ::testing::Test {
public:
    TEST_F(MemoryTests, MemoryLeakDetection) {
        const int ITERATIONS = 1000;
        
        size_t initial_memory = getCurrentMemoryUsage();
        
        for (int i = 0; i < ITERATIONS; i++) {
            TempestGame game;
            game.initialize();
            game.startGame(1);
            
            // Play for a few frames
            for (int frame = 0; frame < 60; frame++) {
                game.updateFrame();
            }
            
            // Game destructor should clean up everything
        }
        
        size_t final_memory = getCurrentMemoryUsage();
        size_t memory_increase = final_memory - initial_memory;
        
        // Allow some increase but detect significant leaks
        EXPECT_LT(memory_increase, 1024 * 1024);  // Less than 1MB growth
    }
    
    TEST_F(MemoryTests, MaximumMemoryUsage) {
        TempestGame game;
        game.initialize();
        
        size_t baseline = getCurrentMemoryUsage();
        
        // Run through multiple waves
        for (int wave = 1; wave <= 16; wave++) {
            game.startGame(wave);
            
            for (int frame = 0; frame < 300; frame++) {
                game.updateFrame();
            }
            
            size_t current_usage = getCurrentMemoryUsage();
            size_t game_memory = current_usage - baseline;
            
            // Game should use reasonable amount of memory
            EXPECT_LT(game_memory, 50 * 1024 * 1024);  // Less than 50MB
        }
    }
    
private:
    size_t getCurrentMemoryUsage() {
        // Platform-specific memory usage query
        #ifdef _WIN32
        PROCESS_MEMORY_COUNTERS pmc;
        if (GetProcessMemoryInfo(GetCurrentProcess(), &pmc, sizeof(pmc))) {
            return pmc.WorkingSetSize;
        }
        #endif
        return 0;
    }
};
```

---

## Verification & Validation

### 1. Reference Comparison Tests
```cpp
class ReferenceTests : public ::testing::Test {
protected:
    // Golden master data from original hardware
    struct ReferenceData {
        std::vector<uint8_t> input_sequence;
        std::vector<uint8_t> expected_output;
        int expected_cycles;
    };
    
public:
    TEST_F(ReferenceTests, MathboxReferenceComparison) {
        // Load reference test vectors from original hardware
        auto reference_tests = loadMathboxReferenceData();
        
        MathboxCoprocessor mathbox;
        
        for (const auto& test : reference_tests) {
            mathbox.reset();
            
            // Apply input sequence
            for (size_t i = 0; i < test.input_sequence.size(); i += 2) {
                uint8_t reg = test.input_sequence[i];
                uint8_t value = test.input_sequence[i + 1];
                mathbox.writeRegister(reg, value);
            }
            
            // Execute and measure
            int start_cycles = mathbox.getCycleCount();
            while (mathbox.isBusy()) {
                mathbox.step();
            }
            int total_cycles = mathbox.getCycleCount() - start_cycles;
            
            // Compare results
            EXPECT_EQ(total_cycles, test.expected_cycles);
            
            for (size_t i = 0; i < test.expected_output.size(); i += 2) {
                uint8_t reg = test.expected_output[i];
                uint8_t expected = test.expected_output[i + 1];
                uint8_t actual = mathbox.readRegister(reg);
                EXPECT_EQ(actual, expected) 
                    << "Register " << std::hex << (int)reg;
            }
        }
    }
    
    TEST_F(ReferenceTests, VectorGraphicsComparison) {
        // Test against known display list outputs
        auto reference_lists = loadVectorReferenceData();
        
        VectorProcessor vg;
        MockVectorDisplay display;
        vg.setDisplay(&display);
        
        for (const auto& test : reference_lists) {
            display.clear();
            
            // Load display list
            for (size_t i = 0; i < test.display_list.size(); i++) {
                vg.memory()->writeWord(0x2000 + i*2, test.display_list[i]);
            }
            
            // Execute
            vg.startExecution(0x2000);
            while (vg.isExecuting()) {
                vg.step();
            }
            
            // Compare rendered output
            auto vectors = display.getRenderedVectors();
            EXPECT_EQ(vectors.size(), test.expected_vectors.size());
            
            for (size_t i = 0; i < vectors.size(); i++) {
                EXPECT_NEAR(vectors[i].x1, test.expected_vectors[i].x1, 1);
                EXPECT_NEAR(vectors[i].y1, test.expected_vectors[i].y1, 1);
                EXPECT_NEAR(vectors[i].x2, test.expected_vectors[i].x2, 1);
                EXPECT_NEAR(vectors[i].y2, test.expected_vectors[i].y2, 1);
                EXPECT_EQ(vectors[i].intensity, test.expected_vectors[i].intensity);
            }
        }
    }
};
```

### 2. Behavioral Validation Tests
```cpp
class BehaviorValidationTests : public ::testing::Test {
public:
    TEST_F(BehaviorValidationTests, EnemyMovementPatterns) {
        // Validate enemy CAM script behaviors match original
        
        EnemyManager enemy_manager;
        
        // Test Flipper NOJUMP behavior
        auto flipper = enemy_manager.spawnFlipper(5, 0x80);
        flipper->setCAMScript(CAMScript::NOJUMP);
        
        Vector3 initial_pos = flipper->getPosition();
        
        // NOJUMP should move straight up at constant speed
        for (int frame = 0; frame < 10; frame++) {
            flipper->update();
        }
        
        Vector3 final_pos = flipper->getPosition();
        
        // Should move only in Y direction
        EXPECT_EQ(final_pos.x, initial_pos.x);
        EXPECT_EQ(final_pos.z, initial_pos.z);
        EXPECT_LT(final_pos.y, initial_pos.y);  // Moving toward player
        
        // Verify constant velocity
        float distance_moved = initial_pos.y - final_pos.y;
        float expected_distance = 10 * FLIPPER_SPEED;  // 10 frames * speed
        EXPECT_NEAR(distance_moved, expected_distance, 1.0f);
    }
    
    TEST_F(BehaviorValidationTests, PulsarBehavior) {
        EnemyManager enemy_manager;
        
        auto pulsar = enemy_manager.spawnPulsar(8, 0x60);
        
        // Pulsar should patrol in power zone
        Vector3 initial_pos = pulsar->getPosition();
        
        for (int frame = 0; frame < 120; frame++) {  // 2 seconds
            pulsar->update();
            
            Vector3 current_pos = pulsar->getPosition();
            
            // Should stay within power zone depth range
            EXPECT_GE(current_pos.y, POWER_ZONE_MIN);
            EXPECT_LE(current_pos.y, POWER_ZONE_MAX);
            
            // Should stay on same line
            EXPECT_EQ(current_pos.x, initial_pos.x);
            EXPECT_EQ(current_pos.z, initial_pos.z);
        }
        
        // Test pulse attack timing
        int pulse_count = 0;
        pulsar->reset();
        
        for (int frame = 0; frame < 600; frame++) {  // 10 seconds
            pulsar->update();
            
            if (pulsar->isPulsing()) {
                pulse_count++;
            }
        }
        
        // Should pulse periodically (exact timing from tuning data)
        EXPECT_GE(pulse_count, 3);  // At least 3 pulses in 10 seconds
        EXPECT_LE(pulse_count, 8);  // No more than 8 pulses
    }
};
```

---

## Test Data and Fixtures

### 1. Reference Data Generation
```cpp
class TestDataGenerator {
public:
    static std::vector<MathboxTestCase> generateMathboxTests() {
        std::vector<MathboxTestCase> tests;
        
        // Multiplication test cases
        for (int16_t a = -100; a <= 100; a += 25) {
            for (int16_t b = -100; b <= 100; b += 25) {
                MathboxTestCase test;
                test.operation = MULTIPLY;
                test.input_a = a;
                test.input_b = b;
                test.expected_result = (int32_t)a * b;
                test.expected_cycles = 46;  // From microcode analysis
                tests.push_back(test);
            }
        }
        
        // Division test cases
        for (int32_t dividend = -10000; dividend <= 10000; dividend += 1000) {
            for (int16_t divisor = -100; divisor <= 100; divisor += 20) {
                if (divisor == 0) continue;  // Skip division by zero
                
                MathboxTestCase test;
                test.operation = DIVIDE;
                test.input_dividend = dividend;
                test.input_divisor = divisor;
                test.expected_quotient = dividend / divisor;
                test.expected_remainder = dividend % divisor;
                test.expected_cycles = 20;  // From microcode analysis
                tests.push_back(test);
            }
        }
        
        return tests;
    }
    
    static std::vector<VectorTestCase> generateVectorTests() {
        std::vector<VectorTestCase> tests;
        
        // Basic shape tests
        VectorTestCase square_test;
        square_test.display_list = {
            0x8000,  // CNTR
            0x4008,  // VCTR right
            0x4100,  // VCTR up  
            0x41F8,  // VCTR left (-8)
            0x4000,  // VCTR down (-8)
            0x2000   // HALT
        };
        
        square_test.expected_vectors = {
            {0, 0, 8, 0, 7},     // Right
            {8, 0, 8, 8, 7},     // Up
            {8, 8, 0, 8, 7},     // Left
            {0, 8, 0, 0, 7}      // Down
        };
        
        tests.push_back(square_test);
        
        // Character rendering tests
        // ... (generate tests for all characters A-Z, 0-9)
        
        return tests;
    }
};
```

### 2. Mock Objects
```cpp
class MockMemoryBus : public MemoryBus {
public:
    MOCK_METHOD(uint8_t, read, (uint16_t address), (override));
    MOCK_METHOD(void, write, (uint16_t address, uint8_t value), (override));
    MOCK_METHOD(uint16_t, readWord, (uint16_t address), (override));
    MOCK_METHOD(void, writeWord, (uint16_t address, uint16_t value), (override));
};

class MockVectorDisplay : public VectorDisplay {
public:
    MOCK_METHOD(void, drawVector, (int16_t x1, int16_t y1, int16_t x2, int16_t y2, 
                                  uint8_t intensity, uint8_t color), (override));
    MOCK_METHOD(void, centerBeam, (), (override));
    MOCK_METHOD(void, setScale, (uint8_t binary, uint8_t linear), (override));
    
    std::vector<RenderedVector> rendered_vectors;
    int center_call_count = 0;
    int vector_call_count = 0;
};

class MockInputManager : public InputManager {
public:
    MOCK_METHOD(int8_t, getRotaryDelta, (), (override));
    MOCK_METHOD(bool, isFirePressed, (), (override));
    MOCK_METHOD(bool, isStartPressed, (), (override));
    MOCK_METHOD(bool, isCoinInserted, (), (override));
    
    void setRotaryDelta(int8_t delta) { rotary_delta = delta; }
    void setFirePressed(bool pressed) { fire_pressed = pressed; }
    
private:
    int8_t rotary_delta = 0;
    bool fire_pressed = false;
};
```

---

## Continuous Integration

### 1. Automated Test Pipeline
```yaml
# GitHub Actions workflow
name: Tempest Test Suite

on: [push, pull_request]

jobs:
  test:
    runs-on: windows-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup MSVC
      uses: microsoft/setup-msbuild@v1
      
    - name: Setup vcpkg
      run: |
        git clone https://github.com/Microsoft/vcpkg.git
        .\vcpkg\bootstrap-vcpkg.bat
        
    - name: Install dependencies
      run: |
        .\vcpkg\vcpkg install gtest:x64-windows
        .\vcpkg\vcpkg install benchmark:x64-windows
        
    - name: Configure CMake
      run: |
        cmake -B build -DCMAKE_TOOLCHAIN_FILE=vcpkg/scripts/buildsystems/vcpkg.cmake
        
    - name: Build
      run: cmake --build build --config Release
      
    - name: Run Unit Tests
      run: .\build\tests\unit_tests.exe --gtest_output=xml:unit_test_results.xml
      
    - name: Run Integration Tests  
      run: .\build\tests\integration_tests.exe --gtest_output=xml:integration_test_results.xml
      
    - name: Run Performance Tests
      run: .\build\tests\performance_tests.exe --benchmark_format=json --benchmark_out=performance_results.json
      
    - name: Upload Test Results
      uses: actions/upload-artifact@v3
      with:
        name: test-results
        path: |
          unit_test_results.xml
          integration_test_results.xml
          performance_results.json
```

### 2. Test Report Generation
```cpp
class TestReporter {
public:
    static void generateHTMLReport(const std::vector<TestResult>& results) {
        std::ofstream report("test_report.html");
        
        report << "<!DOCTYPE html>\n";
        report << "<html><head><title>Tempest Test Report</title></head>\n";
        report << "<body>\n";
        report << "<h1>Tempest Test Results</h1>\n";
        
        // Summary statistics
        int passed = 0, failed = 0;
        for (const auto& result : results) {
            if (result.passed) passed++;
            else failed++;
        }
        
        report << "<div class='summary'>\n";
        report << "<p>Total Tests: " << results.size() << "</p>\n";
        report << "<p>Passed: " << passed << "</p>\n";
        report << "<p>Failed: " << failed << "</p>\n";
        report << "<p>Success Rate: " << (100.0 * passed / results.size()) << "%</p>\n";
        report << "</div>\n";
        
        // Detailed results
        report << "<table border='1'>\n";
        report << "<tr><th>Test Name</th><th>Status</th><th>Duration</th><th>Message</th></tr>\n";
        
        for (const auto& result : results) {
            report << "<tr>\n";
            report << "<td>" << result.name << "</td>\n";
            report << "<td>" << (result.passed ? "PASS" : "FAIL") << "</td>\n";
            report << "<td>" << result.duration_ms << "ms</td>\n";
            report << "<td>" << result.message << "</td>\n";
            report << "</tr>\n";
        }
        
        report << "</table>\n";
        report << "</body></html>\n";
    }
};
```

---

## Conclusion

This comprehensive testing strategy ensures that the Tempest recreation maintains extreme accuracy to the original hardware while providing robust verification of all critical components. The multi-layered approach catches issues at every level, from individual function correctness to full system integration.

**Key Testing Principles**:
- **Accuracy Verification**: Every component tested against original specifications
- **Performance Validation**: Maintains 60+ FPS under all conditions  
- **Regression Prevention**: Comprehensive test suite prevents functionality breaks
- **Continuous Validation**: Automated testing on every code change
- **Reference Comparison**: Golden master testing against known hardware behavior

The test suite provides confidence that the modern recreation will behave identically to the original arcade hardware while taking advantage of contemporary development practices and tools.

**Success Metrics**:
- 95%+ unit test coverage
- 90%+ integration test coverage
- All performance benchmarks met
- Zero memory leaks detected
- All reference tests passing
- Automated CI pipeline green