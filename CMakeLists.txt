cmake_minimum_required(VERSION 3.13)

# Set board to Pico 2 (RP2350)
set(PICO_BOARD pico2 CACHE STRING "Pico board type")

# Board configuration variant (M1, M2)
set(BOARD_VARIANT "M1" CACHE STRING "Board variant: M1 or M2")

# CPU overclock configuration (252, 378, 504 MHz)
# 252 = no overclock (default), 378 = medium, 504 = high
set(CPU_SPEED "378" CACHE STRING "CPU clock in MHz: 252, 378, 504")

# PSRAM speed configuration (84, 100, 133, 166 MHz target)
set(PSRAM_SPEED "133" CACHE STRING "PSRAM max frequency in MHz: 84, 100, 133, 166")

# CPU voltage selection based on speed
if(CPU_SPEED GREATER_EQUAL 504)
    set(CPU_VOLTAGE "VREG_VOLTAGE_1_65")
elseif(CPU_SPEED GREATER_EQUAL 378)
    set(CPU_VOLTAGE "VREG_VOLTAGE_1_60")
else()
    set(CPU_VOLTAGE "VREG_VOLTAGE_1_50")
endif()

message(STATUS "Board: ${BOARD_VARIANT}, CPU: ${CPU_SPEED} MHz, PSRAM: ${PSRAM_SPEED} MHz, Voltage: ${CPU_VOLTAGE}")

include(pico_sdk_import.cmake)

project(murmheretic C CXX ASM)
set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)

pico_sdk_init()

# Add fatfs
add_subdirectory(src/fatfs)
# Add drivers
add_subdirectory(drivers/sdcard)
add_subdirectory(drivers/ps2kbd)
add_subdirectory(drivers/ps2mouse)
add_subdirectory(drivers/usbhid)


# Create a library for other drivers
add_library(drivers
    drivers/HDMI.c
    drivers/psram_init.c
    drivers/psram_allocator.c
)
target_include_directories(drivers PUBLIC drivers)
target_compile_definitions(drivers PRIVATE
    BOARD_${BOARD_VARIANT}
    PSRAM_MAX_FREQ_MHZ=${PSRAM_SPEED}
)
target_compile_options(drivers PRIVATE -Ofast)
target_link_libraries(drivers pico_stdlib hardware_dma hardware_pio hardware_spi)

# =============================================================================
# Heretic Build
# =============================================================================

# Heretic sources
file(GLOB HERETIC_SOURCES src/heretic/*.c)

# Exclude platform specific and replaced files
list(FILTER HERETIC_SOURCES EXCLUDE REGEX ".*doomgeneric_.*\\.c$") # Exclude all doomgeneric_*.c
list(FILTER HERETIC_SOURCES EXCLUDE REGEX ".*i_.*\\.c$") # Exclude all i_*.c
list(FILTER HERETIC_SOURCES EXCLUDE REGEX ".*w_file_stdc\\.c$")
list(FILTER HERETIC_SOURCES EXCLUDE REGEX ".*m_misc\\.c$")
list(FILTER HERETIC_SOURCES EXCLUDE REGEX ".*i_main\\.c$") # Exclude i_main.c
list(FILTER HERETIC_SOURCES EXCLUDE REGEX ".*d_dedicated\\.c$")
list(FILTER HERETIC_SOURCES EXCLUDE REGEX ".*z_native\\.c$")
list(FILTER HERETIC_SOURCES EXCLUDE REGEX ".*midifile\\.c$")

# Add back necessary i_*.c files (from src/heretic which are the doomgeneric ones)
list(APPEND HERETIC_SOURCES
    src/heretic/i_video.c
    src/heretic/i_timer.c
    src/heretic/i_input.c
    src/heretic/i_sound.c
    src/heretic/i_endoom.c
    src/heretic/i_glob.c
    src/heretic/doomgeneric.c
    src/pico/i_picosound.c
    src/pico/i_oplmusic.c
    src/midifile.c
    src/opl/emu8950.c
    src/opl/emuadpcm.c
    src/opl/opl_api.c
    src/opl/opl_pico.c
    src/opl/slot_render.cpp
)

# Remove duplicates
list(REMOVE_DUPLICATES HERETIC_SOURCES)

add_executable(murmheretic
    src/main.c
    src/doomgeneric_rp2350.c
    src/doomgeneric_fatfs/w_file_fatfs.c
    src/doomgeneric_fatfs/m_misc_fatfs.c
    src/doomgeneric_fatfs/stdio_fatfs.c
    src/opl/slot_render_pico.S
    src/pico/audio.c
    ${HERETIC_SOURCES}
)

# Generate PIO header for I2S audio (must be after add_executable)
add_subdirectory(drivers/audio)

target_include_directories(murmheretic PRIVATE
    src
    src/heretic
    src/pico
    src/fatfs
    src/opl
    drivers
    drivers/sdcard
    drivers/ps2mouse
)

target_compile_options(murmheretic PRIVATE -Ofast)

target_compile_definitions(murmheretic PRIVATE
    HERETIC=1
    BOARD_${BOARD_VARIANT}
    CPU_CLOCK_MHZ=${CPU_SPEED}
    CPU_VOLTAGE=${CPU_VOLTAGE}
    PSRAM_MAX_FREQ_MHZ=${PSRAM_SPEED}
    CMAP256
    DOOMGENERIC_RESX=320
    DOOMGENERIC_RESY=240
    FEATURE_SOUND
    USE_PICO_SOUND
    USE_OPL_MUSIC=1
    USE_EMU8950_OPL=1
    NUM_SOUND_CHANNELS=16
    NO_USE_LIBSAMPLERATE
    NO_USE_TIMIDITY
    NO_USE_GUS
    NO_USE_MUSIC_PACKS
    USE_DIRECT_MIDI_LUMP=0
    EMU8950_NO_WAVE_TABLE_MAP=1
    EMU8950_NO_TLL=1
    EMU8950_NO_FLOAT=1
    EMU8950_NO_TIMER=1
    EMU8950_NO_TEST_FLAG=1
    EMU8950_SIMPLER_NOISE=1
    EMU8950_SHORT_NOISE_UPDATE_CHECK=1
    EMU8950_LINEAR_SKIP=1
    EMU8950_LINEAR_END_OF_NOTE_OPTIMIZATION=1
    EMU8950_NO_PERCUSSION_MODE=1
    EMU8950_LINEAR=1
    EMU8950_ASM=1
    EMU8950_SLOT_RENDER=1
    EMU8950_NO_RATECONV=1
    PICO_ON_DEVICE=1
    HERETIC=1
)

target_link_options(murmheretic PRIVATE -Wl,-Map=murmheretic.map)

# Set peripheral pins based on board variant
if(BOARD_VARIANT STREQUAL "M1")
    target_compile_definitions(murmheretic PRIVATE
        # I2S Audio
        PICO_AUDIO_I2S_DATA_PIN=26
        PICO_AUDIO_I2S_CLOCK_PIN_BASE=27
        # SD Card
        SDCARD_PIN_SPI0_CS=5
        SDCARD_PIN_SPI0_SCK=2
        SDCARD_PIN_SPI0_MOSI=3
        SDCARD_PIN_SPI0_MISO=4
    )
elseif(BOARD_VARIANT STREQUAL "M2")
    target_compile_definitions(murmheretic PRIVATE
        # I2S Audio
        PICO_AUDIO_I2S_DATA_PIN=9
        PICO_AUDIO_I2S_CLOCK_PIN_BASE=10
        # SD Card
        SDCARD_PIN_SPI0_CS=5
        SDCARD_PIN_SPI0_SCK=6
        SDCARD_PIN_SPI0_MOSI=7
        SDCARD_PIN_SPI0_MISO=4
    )
endif()

target_link_libraries(murmheretic
    pico_stdlib
    pico_multicore
    hardware_clocks
    hardware_spi
    hardware_dma
    hardware_pio
    hardware_pwm
    hardware_irq
    hardware_interp
    sdcard
    ps2kbd
    ps2mouse
    usbhid
    fatfs
    drivers
)

# Wrap stdio
target_link_options(murmheretic PRIVATE
    -Wl,--wrap=fopen
    -Wl,--wrap=fclose
    -Wl,--wrap=fread
    -Wl,--wrap=fgetc
    -Wl,--wrap=fwrite
    -Wl,--wrap=fseek
    -Wl,--wrap=ftell
    -Wl,--wrap=remove
    -Wl,--wrap=rename
)

# USB stdio
if(USB_HID_ENABLED)
    pico_enable_stdio_usb(murmheretic 0)
    pico_enable_stdio_uart(murmheretic 1)
else()
    pico_enable_stdio_usb(murmheretic 1)
    pico_enable_stdio_uart(murmheretic 0)
endif()

pico_add_extra_outputs(murmheretic)
