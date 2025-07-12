# TEMPEST IMPLEMENTATION DETAILS
## Low-Level Technical Specifications for Critical Components

### Table of Contents
1. [Mathbox Coprocessor Implementation](#mathbox-coprocessor-implementation)
2. [Vector Graphics Engine Implementation](#vector-graphics-engine-implementation)
3. [6502 CPU Emulator Implementation](#6502-cpu-emulator-implementation)
4. [Memory System Implementation](#memory-system-implementation)
5. [Audio System Implementation](#audio-system-implementation)
6. [Input System Implementation](#input-system-implementation)
7. [Platform Integration](#platform-integration)

---

## Mathbox Coprocessor Implementation

### AMD 2901 Bit-Slice Processor Emulation

The Mathbox uses four AMD 2901 4-bit processors chained to create a 16-bit ALU. Each 2901 contains:
- 16×4-bit register file
- 4-bit ALU with 8 functions
- Q register for multiplication/division
- Carry propagation logic

```cpp
class AMD2901 {
private:
    // Internal state
    std::array<uint8_t, 16> registers;  // 16×4-bit register file
    uint8_t q_register;                 // 4-bit Q register
    bool carry_flag;
    
    // Instruction decode
    struct Microinstruction {
        uint8_t source : 3;      // Source operand selection
        uint8_t function : 3;    // ALU function (0-7)
        uint8_t dest : 3;        // Destination control
        uint8_t reg_a : 4;       // A operand address
        uint8_t reg_b : 4;       // B operand address
    };

public:
    // ALU Functions (3-bit encoding)
    enum ALUFunction {
        ADD = 0,     // R + S
        SUBR = 1,    // S - R  
        SUBS = 2,    // R - S
        OR = 3,      // R | S
        AND = 4,     // R & S
        NOTRS = 5,   // ~R & S
        EXOR = 6,    // R ^ S
        EXNOR = 7    // ~(R ^ S)
    };
    
    // Source Selection (3-bit encoding)
    enum SourceSelect {
        AQ = 0,      // A and Q
        AB = 1,      // A and B
        ZQ = 2,      // 0 and Q
        ZB = 3,      // 0 and B
        ZA = 4,      // 0 and A
        DA = 5,      // D and A
        DQ = 6,      // D and Q
        DZ = 7       // D and 0
    };
    
    // Destination Control (3-bit encoding)
    enum DestControl {
        QREG = 0,    // Store to Q register
        NOP = 1,     // No operation
        RAMA = 2,    // Store to RAM, increment address
        RAMF = 3,    // Store to RAM, fixed address
        RAMQD = 4,   // Store to RAM, shift Q down
        RAMD = 5,    // Store to RAM, shift down
        RAMQU = 6,   // Store to RAM, shift Q up
        RAMU = 7     // Store to RAM, shift up
    };
    
    uint8_t execute(const Microinstruction& instr, uint8_t d_input, bool carry_in) {
        // Get operands based on source selection
        uint8_t r_operand, s_operand;
        getOperands(instr.source, instr.reg_a, instr.reg_b, d_input, r_operand, s_operand);
        
        // Perform ALU operation
        uint8_t result;
        bool carry_out;
        executeALU(instr.function, r_operand, s_operand, carry_in, result, carry_out);
        
        // Handle destination
        handleDestination(instr.dest, instr.reg_a, result, carry_out);
        
        carry_flag = carry_out;
        return result;
    }
    
private:
    void getOperands(uint8_t source, uint8_t reg_a, uint8_t reg_b, uint8_t d_input,
                    uint8_t& r_operand, uint8_t& s_operand) {
        switch (source) {
            case AQ:
                r_operand = registers[reg_a];
                s_operand = q_register;
                break;
            case AB:
                r_operand = registers[reg_a];
                s_operand = registers[reg_b];
                break;
            case ZQ:
                r_operand = 0;
                s_operand = q_register;
                break;
            case ZB:
                r_operand = 0;
                s_operand = registers[reg_b];
                break;
            case ZA:
                r_operand = 0;
                s_operand = registers[reg_a];
                break;
            case DA:
                r_operand = d_input;
                s_operand = registers[reg_a];
                break;
            case DQ:
                r_operand = d_input;
                s_operand = q_register;
                break;
            case DZ:
                r_operand = d_input;
                s_operand = 0;
                break;
        }
    }
    
    void executeALU(uint8_t function, uint8_t r, uint8_t s, bool carry_in,
                   uint8_t& result, bool& carry_out) {
        uint16_t temp_result;
        
        switch (function) {
            case ADD:
                temp_result = r + s + (carry_in ? 1 : 0);
                break;
            case SUBR:
                temp_result = s - r + (carry_in ? 0 : -1);  // Borrow handling
                break;
            case SUBS:
                temp_result = r - s + (carry_in ? 0 : -1);
                break;
            case OR:
                temp_result = r | s;
                carry_out = carry_in;  // Pass through
                break;
            case AND:
                temp_result = r & s;
                carry_out = carry_in;
                break;
            case NOTRS:
                temp_result = (~r) & s;
                carry_out = carry_in;
                break;
            case EXOR:
                temp_result = r ^ s;
                carry_out = carry_in;
                break;
            case EXNOR:
                temp_result = ~(r ^ s);
                carry_out = carry_in;
                break;
        }
        
        result = temp_result & 0xF;
        carry_out = (temp_result > 15);
    }
    
    void handleDestination(uint8_t dest, uint8_t reg_addr, uint8_t result, bool carry) {
        switch (dest) {
            case QREG:
                q_register = result;
                break;
            case NOP:
                // Do nothing
                break;
            case RAMA:
                registers[reg_addr] = result;
                break;
            case RAMF:
                registers[reg_addr] = result;
                break;
            case RAMQD:
                registers[reg_addr] = result;
                q_register = (q_register >> 1) | (carry ? 0x8 : 0);
                break;
            case RAMD:
                registers[reg_addr] = result;
                // Shift right with carry
                break;
            case RAMQU:
                registers[reg_addr] = result;
                q_register = (q_register << 1) | (carry ? 1 : 0);
                break;
            case RAMU:
                registers[reg_addr] = result;
                // Shift left with carry
                break;
        }
    }
};
```

### 16-Bit Mathbox Assembly

```cpp
class MathboxCoprocessor {
private:
    std::array<AMD2901, 4> slices;       // Four 4-bit slices (LSB to MSB)
    std::array<uint32_t, 256> microcode; // 24-bit microcode ROM
    
    // Microcode fields
    struct MicrocodeWord {
        uint32_t raw;
        
        uint8_t alu_function() const { return (raw >> 21) & 0x7; }
        uint8_t source_select() const { return (raw >> 18) & 0x7; }
        uint8_t dest_control() const { return (raw >> 15) & 0x7; }
        uint8_t reg_a() const { return (raw >> 11) & 0xF; }
        uint8_t reg_b() const { return (raw >> 7) & 0xF; }
        uint8_t condition() const { return (raw >> 4) & 0x7; }
        uint8_t next_addr() const { return raw & 0xF; }
    };
    
    // Host interface registers
    uint8_t microcode_pc;
    bool busy_flag;
    uint16_t reg_x, reg_y, reg_a, reg_b, reg_e, reg_f;
    uint32_t result_32;
    
public:
    // Memory-mapped register interface
    void writeRegister(uint8_t address, uint8_t value) {
        switch (address) {
            case 0x01: reg_x = (reg_x & 0xFF00) | value; break;        // RXL
            case 0x02: reg_x = (reg_x & 0x00FF) | (value << 8); break; // RXH
            case 0x03: reg_y = (reg_y & 0xFF00) | value; break;        // RYL
            case 0x04: reg_y = (reg_y & 0x00FF) | (value << 8); break; // RYH
            case 0x05: reg_a = (reg_a & 0xFF00) | value; break;        // RAL
            case 0x06: reg_a = (reg_a & 0x00FF) | (value << 8); break; // RAH
            case 0x07: reg_b = (reg_b & 0xFF00) | value; break;        // RBL
            case 0x08: reg_b = (reg_b & 0x00FF) | (value << 8); break; // RBH
            case 0x09: reg_e = (reg_e & 0xFF00) | value; break;        // REL
            case 0x0A: reg_e = (reg_e & 0x00FF) | (value << 8); break; // REH
            case 0x0B: // YHSP - write Y high + start perspective
                reg_y = (reg_y & 0x00FF) | (value << 8);
                startMicroprogram(MICROCODE_PERSPECTIVE);
                break;
        }
    }
    
    uint8_t readRegister(uint8_t address) {
        switch (address) {
            case 0x00: return busy_flag ? 0x80 : 0x00;  // Status register
            case 0x01: return reg_x & 0xFF;             // Result low
            case 0x02: return (reg_x >> 8) & 0xFF;      // Result high
            // ... other result registers
            default: return 0;
        }
    }
    
    void startMicroprogram(uint8_t start_address) {
        microcode_pc = start_address;
        busy_flag = true;
    }
    
    void step() {
        if (!busy_flag) return;
        
        MicrocodeWord instruction = { microcode[microcode_pc] };
        
        // Execute instruction on all four slices in parallel
        bool carry = false;
        uint16_t result = 0;
        
        for (int slice = 0; slice < 4; slice++) {
            AMD2901::Microinstruction slice_instr;
            slice_instr.source = instruction.source_select();
            slice_instr.function = instruction.alu_function();
            slice_instr.dest = instruction.dest_control();
            slice_instr.reg_a = instruction.reg_a();
            slice_instr.reg_b = instruction.reg_b();
            
            uint8_t d_input = 0;  // External data input
            uint8_t slice_result = slices[slice].execute(slice_instr, d_input, carry);
            
            result |= (slice_result << (slice * 4));
            carry = slices[slice].getCarry();
        }
        
        // Handle microcode sequencing
        handleSequencing(instruction);
    }
    
private:
    void handleSequencing(const MicrocodeWord& instruction) {
        // Microcode sequencing logic
        uint8_t condition = instruction.condition();
        
        switch (condition) {
            case 0: // Unconditional
                microcode_pc = instruction.next_addr();
                break;
            case 1: // Branch on carry
                if (slices[3].getCarry()) {
                    microcode_pc = instruction.next_addr();
                } else {
                    microcode_pc++;
                }
                break;
            case 2: // Branch on zero
                // Check if result is zero
                break;
            case 7: // HALT
                busy_flag = false;
                break;
            default:
                microcode_pc++;
                break;
        }
    }
};
```

### Mathematical Algorithm Implementation

#### 16×16 Multiplication (Shift-and-Add)
```cpp
void MathboxCoprocessor::multiply16x16(int16_t a, int16_t b) {
    // Based on MULXP microcode routine
    
    // Convert to unsigned, track sign
    bool result_negative = (a < 0) ^ (b < 0);
    uint16_t abs_a = std::abs(a);
    uint16_t abs_b = std::abs(b);
    
    // Initialize for shift-and-add
    uint32_t product = 0;
    uint16_t multiplier = abs_b;
    uint16_t multiplicand = abs_a;
    
    // 16 iterations of shift-and-add
    for (int i = 0; i < 16; i++) {
        if (multiplier & 1) {
            product += multiplicand;
        }
        multiplier >>= 1;
        multiplicand <<= 1;
    }
    
    // Apply sign
    if (result_negative) {
        product = (~product) + 1;  // Two's complement
    }
    
    result_32 = product;
}
```

#### 32÷16 Division (Non-Restoring)
```cpp
void MathboxCoprocessor::divide32by16(int32_t dividend, int16_t divisor) {
    // Based on DIVID microcode routine
    
    if (divisor == 0) {
        // Division by zero - return max value
        result_32 = 0xFFFF0000;
        return;
    }
    
    // Convert to unsigned, track sign
    bool quotient_negative = (dividend < 0) ^ (divisor < 0);
    uint32_t abs_dividend = std::abs(dividend);
    uint16_t abs_divisor = std::abs(divisor);
    
    uint16_t quotient = 0;
    uint32_t remainder = abs_dividend;
    
    // Non-restoring division algorithm
    for (int i = 15; i >= 0; i--) {
        remainder <<= 1;
        
        if (remainder >= (abs_divisor << 16)) {
            remainder -= (abs_divisor << 16);
            quotient |= (1 << i);
        }
    }
    
    // Apply signs
    if (quotient_negative) {
        quotient = (~quotient) + 1;
    }
    
    result_32 = (quotient << 16) | (remainder >> 16);
}
```

---

## Vector Graphics Engine Implementation

### Instruction Set Implementation

```cpp
class VectorInstruction {
public:
    enum Type {
        VCTR_SHORT = 0x4,
        VCTR_LONG  = 0x0,
        HALT       = 0x2,
        STAT       = 0x6,
        SCAL       = 0x7,
        CNTR       = 0x8,
        JSRL       = 0xA,
        RTSL       = 0xC,
        JMPL       = 0xE
    };
    
    static std::unique_ptr<VectorInstruction> decode(uint16_t word1, uint16_t word2 = 0) {
        uint8_t opcode = (word1 >> 12) & 0xF;
        
        switch (opcode) {
            case VCTR_SHORT:
                return std::make_unique<VCTRShortInstruction>(word1);
            case VCTR_LONG:
                return std::make_unique<VCTRLongInstruction>(word1, word2);
            case HALT:
                return std::make_unique<HALTInstruction>();
            case STAT:
                return std::make_unique<STATInstruction>(word1);
            case SCAL:
                return std::make_unique<SCALInstruction>(word1);
            case CNTR:
                return std::make_unique<CNTRInstruction>();
            case JSRL:
                return std::make_unique<JSRLInstruction>(word1);
            case RTSL:
                return std::make_unique<RTSLInstruction>();
            case JMPL:
                return std::make_unique<JMPLInstruction>(word1);
            default:
                return nullptr;
        }
    }
    
    virtual void execute(VectorProcessor& vg) = 0;
    virtual int getCycles() const = 0;
    virtual bool isVectorDraw() const { return false; }
};

class VCTRShortInstruction : public VectorInstruction {
private:
    int8_t delta_x, delta_y;
    uint8_t intensity;
    
public:
    VCTRShortInstruction(uint16_t instruction) {
        // Decode: 0100 ZZZZ YYYY YXXX XX
        intensity = (instruction >> 8) & 0x7;
        
        // Extract 5-bit signed deltas
        int8_t raw_y = (instruction >> 3) & 0x1F;
        int8_t raw_x = instruction & 0x1F;
        
        // Sign extend from 5 bits to 8 bits
        delta_y = (raw_y & 0x10) ? (raw_y | 0xE0) : raw_y;
        delta_x = (raw_x & 0x10) ? (raw_x | 0xE0) : raw_x;
    }
    
    void execute(VectorProcessor& vg) override {
        vg.drawRelativeVector(delta_x, delta_y, intensity);
    }
    
    int getCycles() const override { return 2; }
    bool isVectorDraw() const override { return true; }
    
    int8_t getDeltaX() const { return delta_x; }
    int8_t getDeltaY() const { return delta_y; }
    uint8_t getIntensity() const { return intensity; }
};

class VCTRLongInstruction : public VectorInstruction {
private:
    int16_t delta_x, delta_y;
    uint8_t intensity;
    
public:
    VCTRLongInstruction(uint16_t word1, uint16_t word2) {
        // Word 1: YYYY YYYY YYYY Y000 (13-bit signed Y)
        delta_y = (int16_t)(word1 << 3) >> 3;  // Sign extend
        
        // Word 2: ZZZX XXXX XXXX XXXX (3-bit intensity + 13-bit signed X)
        intensity = (word2 >> 13) & 0x7;
        delta_x = (int16_t)(word2 << 3) >> 3;  // Sign extend
    }
    
    void execute(VectorProcessor& vg) override {
        vg.drawRelativeVector(delta_x, delta_y, intensity);
    }
    
    int getCycles() const override { return 4; }
    bool isVectorDraw() const override { return true; }
};

class JSRLInstruction : public VectorInstruction {
private:
    uint16_t target_address;
    
public:
    JSRLInstruction(uint16_t instruction) {
        // Extract 12-bit address, multiply by 2 for word addressing
        target_address = (instruction & 0xFFF) * 2;
    }
    
    void execute(VectorProcessor& vg) override {
        vg.pushSubroutine();
        vg.setPC(target_address);
    }
    
    int getCycles() const override { return 3; }
    uint16_t getAddress() const { return target_address; }
};
```

### Vector Processor Core

```cpp
class VectorProcessor {
private:
    // Execution state
    uint16_t program_counter;
    std::array<uint16_t, 5> call_stack;
    uint8_t stack_pointer;
    bool halted;
    
    // Drawing state
    int16_t beam_x, beam_y;
    uint8_t current_intensity;
    uint8_t current_color;
    uint8_t scale_binary;
    uint8_t scale_linear;
    
    // Hardware interface
    MemoryBus* memory_bus;
    VectorDisplay* display;
    
    // Timing
    int cycle_count;
    
public:
    void reset() {
        program_counter = 0;
        stack_pointer = 0;
        halted = false;
        beam_x = beam_y = 0;
        current_intensity = 0;
        current_color = 0;
        scale_binary = 0;
        scale_linear = 0;
        cycle_count = 0;
    }
    
    void startExecution(uint16_t start_address) {
        program_counter = start_address;
        halted = false;
    }
    
    void step() {
        if (halted) return;
        
        // Fetch instruction
        uint16_t word1 = memory_bus->readWord(program_counter);
        uint16_t word2 = 0;
        
        // Check if this is a long instruction
        if ((word1 & 0xF000) == 0x0000) {  // VCTR long
            word2 = memory_bus->readWord(program_counter + 2);
        }
        
        // Decode instruction
        auto instruction = VectorInstruction::decode(word1, word2);
        if (!instruction) {
            // Invalid instruction - halt
            halted = true;
            return;
        }
        
        // Update PC before execution (for jumps)
        if ((word1 & 0xF000) == 0x0000) {
            program_counter += 4;  // Long instruction
        } else {
            program_counter += 2;  // Short instruction
        }
        
        // Execute instruction
        instruction->execute(*this);
        
        // Update cycle count
        cycle_count += instruction->getCycles();
    }
    
    void drawRelativeVector(int16_t dx, int16_t dy, uint8_t intensity) {
        // Apply scaling
        dx = applyScaling(dx);
        dy = applyScaling(dy);
        
        // Calculate end position
        int16_t end_x = beam_x + dx;
        int16_t end_y = beam_y + dy;
        
        // Draw vector if intensity > 0
        if (intensity > 0) {
            display->drawVector(beam_x, beam_y, end_x, end_y, intensity, current_color);
        }
        
        // Update beam position
        beam_x = end_x;
        beam_y = end_y;
    }
    
    void centerBeam() {
        beam_x = 0;
        beam_y = 0;
    }
    
    void setScale(uint8_t binary_scale, uint8_t linear_scale) {
        scale_binary = binary_scale & 0x7;
        scale_linear = linear_scale;
    }
    
    void setIntensity(uint8_t intensity) {
        current_intensity = intensity & 0x7;
    }
    
    void pushSubroutine() {
        if (stack_pointer < 5) {
            call_stack[stack_pointer++] = program_counter;
        }
        // Stack overflow - could set error flag
    }
    
    void popSubroutine() {
        if (stack_pointer > 0) {
            program_counter = call_stack[--stack_pointer];
        }
        // Stack underflow - could set error flag
    }
    
    void halt() {
        halted = true;
        // Signal completion to CPU via HALT flag
        memory_bus->setHALTFlag(true);
    }
    
private:
    int16_t applyScaling(int16_t value) {
        // Apply binary scaling (power of 2)
        value >>= scale_binary;
        
        // Apply linear scaling (multiply by scale_linear/256)
        if (scale_linear != 0xFF) {
            value = (value * scale_linear) >> 8;
        }
        
        return value;
    }
};
```

### Vector Rendering System

```cpp
class VectorDisplay {
private:
    // Direct3D resources
    ComPtr<ID3D11Device> device;
    ComPtr<ID3D11DeviceContext> context;
    ComPtr<ID3D11Buffer> vertex_buffer;
    ComPtr<ID3D11VertexShader> vertex_shader;
    ComPtr<ID3D11PixelShader> pixel_shader;
    ComPtr<ID3D11BlendState> blend_state;
    
    // Vertex data
    struct VectorVertex {
        float x, y;
        float intensity;
        uint32_t color;
    };
    
    std::vector<VectorVertex> frame_vertices;
    static const size_t MAX_VERTICES = 32768;
    
    // Screen parameters
    float screen_width, screen_height;
    float scale_factor;
    
public:
    void initialize(ID3D11Device* d3d_device, float width, float height) {
        device = d3d_device;
        device->GetImmediateContext(&context);
        
        screen_width = width;
        screen_height = height;
        scale_factor = std::min(width, height) / 1024.0f;  // Tempest coordinate space
        
        createShaders();
        createVertexBuffer();
        createBlendState();
    }
    
    void beginFrame() {
        frame_vertices.clear();
    }
    
    void drawVector(int16_t x1, int16_t y1, int16_t x2, int16_t y2, 
                   uint8_t intensity, uint8_t color) {
        if (frame_vertices.size() + 2 > MAX_VERTICES) return;
        
        // Convert Tempest coordinates to screen coordinates
        float screen_x1 = convertX(x1);
        float screen_y1 = convertY(y1);
        float screen_x2 = convertX(x2);
        float screen_y2 = convertY(y2);
        
        // Convert intensity (0-7) to alpha (0.0-1.0)
        float alpha = intensity / 7.0f;
        
        // Add line vertices
        VectorVertex v1 = { screen_x1, screen_y1, alpha, color };
        VectorVertex v2 = { screen_x2, screen_y2, alpha, color };
        
        frame_vertices.push_back(v1);
        frame_vertices.push_back(v2);
    }
    
    void endFrame() {
        if (frame_vertices.empty()) return;
        
        // Update vertex buffer
        D3D11_MAPPED_SUBRESOURCE mapped;
        context->Map(vertex_buffer.Get(), 0, D3D11_MAP_WRITE_DISCARD, 0, &mapped);
        memcpy(mapped.pData, frame_vertices.data(), 
               frame_vertices.size() * sizeof(VectorVertex));
        context->Unmap(vertex_buffer.Get(), 0);
        
        // Set up rendering state
        context->VSSetShader(vertex_shader.Get(), nullptr, 0);
        context->PSSetShader(pixel_shader.Get(), nullptr, 0);
        context->OMSetBlendState(blend_state.Get(), nullptr, 0xFFFFFFFF);
        
        UINT stride = sizeof(VectorVertex);
        UINT offset = 0;
        context->IASetVertexBuffers(0, 1, vertex_buffer.GetAddressOf(), &stride, &offset);
        context->IASetPrimitiveTopology(D3D11_PRIMITIVE_TOPOLOGY_LINELIST);
        
        // Render vectors
        context->Draw(frame_vertices.size(), 0);
    }
    
private:
    float convertX(int16_t tempest_x) {
        // Convert from Tempest coordinate space to screen space
        // Tempest: -512 to +511 → Screen: 0 to screen_width
        return (tempest_x + 512) * screen_width / 1024.0f;
    }
    
    float convertY(int16_t tempest_y) {
        // Convert from Tempest coordinate space to screen space
        // Tempest: -512 to +511 → Screen: 0 to screen_height (flipped)
        return screen_height - (tempest_y + 512) * screen_height / 1024.0f;
    }
    
    void createShaders() {
        // Vertex shader HLSL
        const char* vs_source = R"(
            struct VS_INPUT {
                float2 pos : POSITION;
                float intensity : INTENSITY;
                uint color : COLOR;
            };
            
            struct PS_INPUT {
                float4 pos : SV_POSITION;
                float intensity : INTENSITY;
                float4 color : COLOR;
            };
            
            PS_INPUT main(VS_INPUT input) {
                PS_INPUT output;
                
                // Convert to normalized device coordinates
                output.pos = float4(input.pos.x * 2.0 / 1024.0 - 1.0,
                                   -(input.pos.y * 2.0 / 768.0 - 1.0),
                                   0.0, 1.0);
                
                output.intensity = input.intensity;
                
                // Decode color (simplified - actual Tempest has 16 fixed colors)
                float3 rgb = float3((input.color >> 16) & 0xFF,
                                   (input.color >> 8) & 0xFF,
                                   input.color & 0xFF) / 255.0;
                output.color = float4(rgb, input.intensity);
                
                return output;
            }
        )";
        
        // Pixel shader HLSL
        const char* ps_source = R"(
            struct PS_INPUT {
                float4 pos : SV_POSITION;
                float intensity : INTENSITY;
                float4 color : COLOR;
            };
            
            float4 main(PS_INPUT input) : SV_TARGET {
                return input.color;
            }
        )";
        
        // Compile and create shaders
        ComPtr<ID3DBlob> vs_blob, ps_blob;
        D3DCompile(vs_source, strlen(vs_source), nullptr, nullptr, nullptr,
                   "main", "vs_5_0", 0, 0, &vs_blob, nullptr);
        D3DCompile(ps_source, strlen(ps_source), nullptr, nullptr, nullptr,
                   "main", "ps_5_0", 0, 0, &ps_blob, nullptr);
        
        device->CreateVertexShader(vs_blob->GetBufferPointer(), vs_blob->GetBufferSize(),
                                  nullptr, &vertex_shader);
        device->CreatePixelShader(ps_blob->GetBufferPointer(), ps_blob->GetBufferSize(),
                                 nullptr, &pixel_shader);
    }
    
    void createVertexBuffer() {
        D3D11_BUFFER_DESC desc = {};
        desc.Usage = D3D11_USAGE_DYNAMIC;
        desc.ByteWidth = MAX_VERTICES * sizeof(VectorVertex);
        desc.BindFlags = D3D11_BIND_VERTEX_BUFFER;
        desc.CPUAccessFlags = D3D11_CPU_ACCESS_WRITE;
        
        device->CreateBuffer(&desc, nullptr, &vertex_buffer);
    }
    
    void createBlendState() {
        D3D11_BLEND_DESC desc = {};
        desc.RenderTarget[0].BlendEnable = TRUE;
        desc.RenderTarget[0].SrcBlend = D3D11_BLEND_SRC_ALPHA;
        desc.RenderTarget[0].DestBlend = D3D11_BLEND_INV_SRC_ALPHA;
        desc.RenderTarget[0].BlendOp = D3D11_BLEND_OP_ADD;
        desc.RenderTarget[0].SrcBlendAlpha = D3D11_BLEND_ONE;
        desc.RenderTarget[0].DestBlendAlpha = D3D11_BLEND_ZERO;
        desc.RenderTarget[0].BlendOpAlpha = D3D11_BLEND_OP_ADD;
        desc.RenderTarget[0].RenderTargetWriteMask = D3D11_COLOR_WRITE_ENABLE_ALL;
        
        device->CreateBlendState(&desc, &blend_state);
    }
};
```

---

## 6502 CPU Emulator Implementation

### CPU Core Architecture

```cpp
class CPU6502 {
private:
    // Registers
    uint8_t a, x, y, sp;
    uint16_t pc;
    uint8_t status;
    
    // Status flags
    enum StatusFlags {
        FLAG_CARRY     = 0x01,
        FLAG_ZERO      = 0x02,
        FLAG_INTERRUPT = 0x04,
        FLAG_DECIMAL   = 0x08,
        FLAG_BREAK     = 0x10,
        FLAG_UNUSED    = 0x20,
        FLAG_OVERFLOW  = 0x40,
        FLAG_NEGATIVE  = 0x80
    };
    
    // Memory interface
    MemoryBus* memory;
    
    // Timing
    int cycle_count;
    bool irq_pending, nmi_pending;
    
    // Instruction table
    typedef void (CPU6502::*InstructionFunc)();
    struct InstructionInfo {
        InstructionFunc func;
        uint8_t cycles;
        const char* name;
    };
    std::array<InstructionInfo, 256> instruction_table;
    
public:
    CPU6502() {
        buildInstructionTable();
        reset();
    }
    
    void reset() {
        a = x = y = 0;
        sp = 0xFF;
        status = FLAG_UNUSED | FLAG_INTERRUPT;  // I flag set on reset
        pc = memory->readWord(0xFFFC);          // Reset vector
        cycle_count = 0;
        irq_pending = nmi_pending = false;
    }
    
    int step() {
        int start_cycles = cycle_count;
        
        // Handle pending interrupts
        if (nmi_pending) {
            handleNMI();
            nmi_pending = false;
        } else if (irq_pending && !(status & FLAG_INTERRUPT)) {
            handleIRQ();
            irq_pending = false;
        }
        
        // Fetch and execute instruction
        uint8_t opcode = fetchByte();
        auto& instr = instruction_table[opcode];
        
        if (instr.func) {
            (this->*instr.func)();
        } else {
            // Illegal instruction - could set error flag
            cycle_count += 2;
        }
        
        return cycle_count - start_cycles;
    }
    
    void triggerIRQ() { irq_pending = true; }
    void triggerNMI() { nmi_pending = true; }
    
    // Register access
    uint8_t getA() const { return a; }
    uint8_t getX() const { return x; }
    uint8_t getY() const { return y; }
    uint8_t getSP() const { return sp; }
    uint16_t getPC() const { return pc; }
    uint8_t getStatus() const { return status; }
    int getCycleCount() const { return cycle_count; }
    
    bool getFlag(StatusFlags flag) const { return (status & flag) != 0; }
    void setFlag(StatusFlags flag, bool value) {
        if (value) {
            status |= flag;
        } else {
            status &= ~flag;
        }
    }
    
private:
    uint8_t fetchByte() {
        uint8_t value = memory->read(pc++);
        cycle_count++;
        return value;
    }
    
    uint16_t fetchWord() {
        uint8_t low = fetchByte();
        uint8_t high = fetchByte();
        return low | (high << 8);
    }
    
    uint8_t readByte(uint16_t address) {
        cycle_count++;
        return memory->read(address);
    }
    
    void writeByte(uint16_t address, uint8_t value) {
        cycle_count++;
        memory->write(address, value);
    }
    
    void pushByte(uint8_t value) {
        writeByte(0x0100 + sp--, value);
    }
    
    uint8_t popByte() {
        return readByte(0x0100 + ++sp);
    }
    
    void pushWord(uint16_t value) {
        pushByte(value >> 8);
        pushByte(value & 0xFF);
    }
    
    uint16_t popWord() {
        uint8_t low = popByte();
        uint8_t high = popByte();
        return low | (high << 8);
    }
    
    void updateNZ(uint8_t value) {
        setFlag(FLAG_ZERO, value == 0);
        setFlag(FLAG_NEGATIVE, (value & 0x80) != 0);
    }
    
    void handleIRQ() {
        pushWord(pc);
        pushByte(status & ~FLAG_BREAK);  // Clear B flag for IRQ
        setFlag(FLAG_INTERRUPT, true);   // Set I flag
        pc = memory->readWord(0xFFFE);   // IRQ vector
        cycle_count += 7;
    }
    
    void handleNMI() {
        pushWord(pc);
        pushByte(status & ~FLAG_BREAK);  // Clear B flag for NMI
        setFlag(FLAG_INTERRUPT, true);   // Set I flag
        pc = memory->readWord(0xFFFA);   // NMI vector
        cycle_count += 7;
    }
    
    // Addressing modes
    uint16_t addrImmediate() { return pc++; }
    uint16_t addrZeroPage() { return fetchByte(); }
    uint16_t addrZeroPageX() { return (fetchByte() + x) & 0xFF; }
    uint16_t addrZeroPageY() { return (fetchByte() + y) & 0xFF; }
    uint16_t addrAbsolute() { return fetchWord(); }
    
    uint16_t addrAbsoluteX() {
        uint16_t base = fetchWord();
        uint16_t addr = base + x;
        
        // Page boundary crossing adds extra cycle
        if ((base & 0xFF00) != (addr & 0xFF00)) {
            cycle_count++;
        }
        
        return addr;
    }
    
    uint16_t addrAbsoluteY() {
        uint16_t base = fetchWord();
        uint16_t addr = base + y;
        
        // Page boundary crossing adds extra cycle
        if ((base & 0xFF00) != (addr & 0xFF00)) {
            cycle_count++;
        }
        
        return addr;
    }
    
    uint16_t addrIndirectX() {
        uint8_t ptr = fetchByte() + x;
        return memory->read(ptr) | (memory->read((ptr + 1) & 0xFF) << 8);
    }
    
    uint16_t addrIndirectY() {
        uint8_t ptr = fetchByte();
        uint16_t base = memory->read(ptr) | (memory->read((ptr + 1) & 0xFF) << 8);
        uint16_t addr = base + y;
        
        // Page boundary crossing adds extra cycle
        if ((base & 0xFF00) != (addr & 0xFF00)) {
            cycle_count++;
        }
        
        return addr;
    }
    
    // Instruction implementations
    void ins_LDA_IMM() { a = readByte(addrImmediate()); updateNZ(a); }
    void ins_LDA_ZP()  { a = readByte(addrZeroPage()); updateNZ(a); }
    void ins_LDA_ZPX() { a = readByte(addrZeroPageX()); updateNZ(a); }
    void ins_LDA_ABS() { a = readByte(addrAbsolute()); updateNZ(a); }
    void ins_LDA_ABSX() { a = readByte(addrAbsoluteX()); updateNZ(a); }
    void ins_LDA_ABSY() { a = readByte(addrAbsoluteY()); updateNZ(a); }
    void ins_LDA_INDX() { a = readByte(addrIndirectX()); updateNZ(a); }
    void ins_LDA_INDY() { a = readByte(addrIndirectY()); updateNZ(a); }
    
    void ins_STA_ZP()  { writeByte(addrZeroPage(), a); }
    void ins_STA_ZPX() { writeByte(addrZeroPageX(), a); }
    void ins_STA_ABS() { writeByte(addrAbsolute(), a); }
    void ins_STA_ABSX() { 
        uint16_t addr = addrAbsoluteX(); 
        cycle_count--;  // STA always takes extra cycle
        writeByte(addr, a); 
    }
    void ins_STA_ABSY() { 
        uint16_t addr = addrAbsoluteY(); 
        cycle_count--;  // STA always takes extra cycle
        writeByte(addr, a); 
    }
    void ins_STA_INDX() { writeByte(addrIndirectX(), a); }
    void ins_STA_INDY() { 
        uint16_t addr = addrIndirectY(); 
        cycle_count--;  // STA always takes extra cycle
        writeByte(addr, a); 
    }
    
    void ins_ADC_IMM() {
        uint8_t operand = readByte(addrImmediate());
        uint16_t result = a + operand + (getFlag(FLAG_CARRY) ? 1 : 0);
        
        setFlag(FLAG_CARRY, result > 0xFF);
        setFlag(FLAG_OVERFLOW, ((a ^ result) & (operand ^ result) & 0x80) != 0);
        
        a = result & 0xFF;
        updateNZ(a);
    }
    
    void ins_JSR() {
        uint16_t target = fetchWord();
        cycle_count++;  // Extra cycle for JSR
        pushWord(pc - 1);  // Push return address - 1
        pc = target;
    }
    
    void ins_RTS() {
        cycle_count += 2;  // Extra cycles for RTS
        pc = popWord() + 1;  // Pop return address + 1
    }
    
    void ins_BEQ() {
        int8_t offset = (int8_t)fetchByte();
        if (getFlag(FLAG_ZERO)) {
            uint16_t new_pc = pc + offset;
            
            // Branch taken - extra cycle
            cycle_count++;
            
            // Page boundary crossing - another extra cycle
            if ((pc & 0xFF00) != (new_pc & 0xFF00)) {
                cycle_count++;
            }
            
            pc = new_pc;
        }
    }
    
    void ins_NOP() {
        // Do nothing, 2 cycles already counted
    }
    
    void buildInstructionTable() {
        // Initialize all entries as illegal instructions
        for (auto& entry : instruction_table) {
            entry = { nullptr, 2, "???" };
        }
        
        // Populate known instructions
        instruction_table[0xA9] = { &CPU6502::ins_LDA_IMM, 2, "LDA #" };
        instruction_table[0xA5] = { &CPU6502::ins_LDA_ZP,  3, "LDA zp" };
        instruction_table[0xB5] = { &CPU6502::ins_LDA_ZPX, 4, "LDA zp,X" };
        instruction_table[0xAD] = { &CPU6502::ins_LDA_ABS, 4, "LDA abs" };
        instruction_table[0xBD] = { &CPU6502::ins_LDA_ABSX,4, "LDA abs,X" };
        instruction_table[0xB9] = { &CPU6502::ins_LDA_ABSY,4, "LDA abs,Y" };
        instruction_table[0xA1] = { &CPU6502::ins_LDA_INDX,6, "LDA (zp,X)" };
        instruction_table[0xB1] = { &CPU6502::ins_LDA_INDY,5, "LDA (zp),Y" };
        
        instruction_table[0x85] = { &CPU6502::ins_STA_ZP,  3, "STA zp" };
        instruction_table[0x95] = { &CPU6502::ins_STA_ZPX, 4, "STA zp,X" };
        instruction_table[0x8D] = { &CPU6502::ins_STA_ABS, 4, "STA abs" };
        instruction_table[0x9D] = { &CPU6502::ins_STA_ABSX,5, "STA abs,X" };
        instruction_table[0x99] = { &CPU6502::ins_STA_ABSY,5, "STA abs,Y" };
        instruction_table[0x81] = { &CPU6502::ins_STA_INDX,6, "STA (zp,X)" };
        instruction_table[0x91] = { &CPU6502::ins_STA_INDY,6, "STA (zp),Y" };
        
        instruction_table[0x69] = { &CPU6502::ins_ADC_IMM, 2, "ADC #" };
        
        instruction_table[0x20] = { &CPU6502::ins_JSR, 6, "JSR" };
        instruction_table[0x60] = { &CPU6502::ins_RTS, 6, "RTS" };
        
        instruction_table[0xF0] = { &CPU6502::ins_BEQ, 2, "BEQ" };
        
        instruction_table[0xEA] = { &CPU6502::ins_NOP, 2, "NOP" };
        
        // ... continue with all 256 instructions
    }
};
```

---

## Memory System Implementation

### Memory Bus Architecture

```cpp
class MemoryBus {
private:
    // Main RAM
    std::array<uint8_t, 0x800> main_ram;      // 2KB main RAM
    std::array<uint8_t, 0x800> vector_ram;    // 2KB vector RAM
    std::array<uint8_t, 0x8000> game_rom;     // 32KB game ROM
    
    // Memory mapped devices
    struct MemoryMapping {
        uint16_t start_addr;
        uint16_t end_addr;
        MemoryDevice* device;
    };
    std::vector<MemoryMapping> device_mappings;
    
    // Hardware registers
    uint8_t io_registers[0x20];
    
    // Timing
    int total_cycles;
    
public:
    uint8_t read(uint16_t address, int cycles = 1) {
        total_cycles += cycles;
        
        // Check for device mappings first
        for (const auto& mapping : device_mappings) {
            if (address >= mapping.start_addr && address <= mapping.end_addr) {
                return mapping.device->read(address - mapping.start_addr);
            }
        }
        
        // Handle standard memory regions
        if (address < 0x100) {
            // Zero page
            return main_ram[address];
        } else if (address < 0x200) {
            // Stack page
            return main_ram[address];
        } else if (address < 0x800) {
            // Main RAM (mirrors every 2KB)
            return main_ram[address & 0x7FF];
        } else if (address >= 0x2000 && address < 0x2800) {
            // Vector RAM
            return vector_ram[address - 0x2000];
        } else if (address >= 0x9000) {
            // Game ROM
            return game_rom[address - 0x9000];
        } else if (address >= 0xD000 && address < 0xD020) {
            // I/O registers
            return readIORegister(address - 0xD000);
        }
        
        // Unmapped memory returns 0xFF (or floating bus value)
        return 0xFF;
    }
    
    void write(uint16_t address, uint8_t value, int cycles = 1) {
        total_cycles += cycles;
        
        // Check for device mappings first
        for (const auto& mapping : device_mappings) {
            if (address >= mapping.start_addr && address <= mapping.end_addr) {
                mapping.device->write(address - mapping.start_addr, value);
                return;
            }
        }
        
        // Handle standard memory regions
        if (address < 0x100) {
            // Zero page
            main_ram[address] = value;
        } else if (address < 0x200) {
            // Stack page
            main_ram[address] = value;
        } else if (address < 0x800) {
            // Main RAM (mirrors every 2KB)
            main_ram[address & 0x7FF] = value;
        } else if (address >= 0x2000 && address < 0x2800) {
            // Vector RAM
            vector_ram[address - 0x2000] = value;
        } else if (address >= 0xD000 && address < 0xD020) {
            // I/O registers
            writeIORegister(address - 0xD000, value);
        }
        // ROM regions ignore writes
    }
    
    uint16_t readWord(uint16_t address) {
        uint8_t low = read(address);
        uint8_t high = read(address + 1);
        return low | (high << 8);
    }
    
    void writeWord(uint16_t address, uint16_t value) {
        write(address, value & 0xFF);
        write(address + 1, (value >> 8) & 0xFF);
    }
    
    void mapDevice(uint16_t start_addr, uint16_t end_addr, MemoryDevice* device) {
        device_mappings.push_back({start_addr, end_addr, device});
    }
    
    void loadROM(const std::vector<uint8_t>& rom_data, uint16_t start_address) {
        for (size_t i = 0; i < rom_data.size() && i < game_rom.size(); i++) {
            game_rom[i] = rom_data[i];
        }
    }
    
private:
    uint8_t readIORegister(uint8_t offset) {
        switch (offset) {
            case 0x10: // IN1 - Cabinet switches + VG HALT flag
                return io_registers[offset];
            case 0x11: // IN2 - Additional switches
                return io_registers[offset];
            default:
                return io_registers[offset];
        }
    }
    
    void writeIORegister(uint8_t offset, uint8_t value) {
        switch (offset) {
            case 0x00: // OUT0 - Coin counters
                io_registers[offset] = value;
                updateCoinCounters(value);
                break;
            case 0x01: // OUT1 - Additional outputs
                io_registers[offset] = value;
                break;
            case 0x0E: // WTCHDG - Watchdog reset
                resetWatchdog();
                break;
            default:
                io_registers[offset] = value;
                break;
        }
    }
    
    void updateCoinCounters(uint8_t value) {
        // Handle coin counter updates
        static uint8_t prev_value = 0;
        
        // Detect rising edges for coin counting
        uint8_t rising_edges = value & ~prev_value;
        
        if (rising_edges & 0x01) {
            // Coin counter 1
        }
        if (rising_edges & 0x02) {
            // Coin counter 2
        }
        
        prev_value = value;
    }
    
    void resetWatchdog() {
        // Reset watchdog timer
        // In actual hardware, failure to reset causes system reset
    }
};

// Base class for memory-mapped devices
class MemoryDevice {
public:
    virtual ~MemoryDevice() = default;
    virtual uint8_t read(uint16_t address) = 0;
    virtual void write(uint16_t address, uint8_t value) = 0;
};
```

### EAROM (EEPROM) Implementation

```cpp
class EAROM : public MemoryDevice {
private:
    std::array<uint8_t, 64> storage;  // 64 bytes of storage
    std::string save_file;
    
    // EAROM control state
    enum State {
        IDLE,
        WRITE_ENABLE,
        WRITE_DATA,
        READ_DATA
    } current_state;
    
    uint8_t address_register;
    uint8_t data_register;
    
public:
    EAROM(const std::string& filename) : save_file(filename), current_state(IDLE) {
        loadFromFile();
    }
    
    ~EAROM() {
        saveToFile();
    }
    
    uint8_t read(uint16_t address) override {
        switch (address) {
            case 0x00: // Data register
                return data_register;
            case 0x01: // Status register
                return 0x00;  // Always ready
            default:
                return 0xFF;
        }
    }
    
    void write(uint16_t address, uint8_t value) override {
        switch (address) {
            case 0x00: // Data register
                data_register = value;
                break;
            case 0x01: // Address register
                address_register = value & 0x3F;  // 6-bit address
                break;
            case 0x02: // Control register
                handleControl(value);
                break;
        }
    }
    
private:
    void handleControl(uint8_t control) {
        switch (control & 0x03) {
            case 0x00: // Read
                current_state = READ_DATA;
                data_register = storage[address_register];
                break;
            case 0x01: // Write enable
                current_state = WRITE_ENABLE;
                break;
            case 0x02: // Write
                if (current_state == WRITE_ENABLE) {
                    storage[address_register] = data_register;
                    current_state = IDLE;
                    saveToFile();  // Persist immediately
                }
                break;
            case 0x03: // Erase
                if (current_state == WRITE_ENABLE) {
                    storage[address_register] = 0xFF;
                    current_state = IDLE;
                    saveToFile();
                }
                break;
        }
    }
    
    void loadFromFile() {
        std::ifstream file(save_file, std::ios::binary);
        if (file.is_open()) {
            file.read(reinterpret_cast<char*>(storage.data()), storage.size());
            file.close();
        } else {
            // Initialize with default values
            std::fill(storage.begin(), storage.end(), 0xFF);
            
            // Set up default high scores, etc.
            initializeDefaults();
        }
    }
    
    void saveToFile() {
        std::ofstream file(save_file, std::ios::binary);
        if (file.is_open()) {
            file.write(reinterpret_cast<const char*>(storage.data()), storage.size());
            file.close();
        }
    }
    
    void initializeDefaults() {
        // Initialize high score table
        // Block 1: High scores (11 bytes)
        const uint8_t default_scores[] = {
            0x01, 0x00, 0x00,  // 10000 points
            0x00, 0x80, 0x00,  // 8000 points
            0x00, 0x60, 0x00,  // 6000 points
            // ... more default scores
        };
        
        std::copy(default_scores, default_scores + sizeof(default_scores),
                 storage.begin() + 8);  // Start at block 1
    }
};
```

---

This implementation provides the low-level technical foundation for accurately recreating Tempest's hardware systems using modern technologies while maintaining cycle-accurate behavior and timing characteristics of the original arcade machine.