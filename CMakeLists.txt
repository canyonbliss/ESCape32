cmake_minimum_required(VERSION 3.15)
set(CMAKE_SYSTEM_NAME Generic)

#set(CMAKE_C_COMPILER arm-none-eabi-gcc)

option(BUILD_DEPS "enable building of libopencm3" OFF)

if(DEFINED ENV{CC})
    set(CMAKE_C_COMPILER $ENV{CC})
    include_directories("/usr/arm-none-eabi/include")
else()
    set(CMAKE_C_COMPILER arm-none-eabi-gcc)
endif()

if(DEFINED ENV{CXX})
    set(CMAKE_CXX_COMPILER $ENV{CXX})
else()
    set(CMAKE_CXX_COMPILER arm-none-eabi-g++)
endif()

set(CMAKE_C_COMPILER_WORKS 1)
set(CMAKE_C_FLAGS "-Os -g")
set(CMAKE_C_FLAGS_DEBUG "-Og")
project(ESCape32 C)

if(BUILD_DEPS)
	include(ExternalProject)
	set(OPENCM3_TARGETS "stm32/f0 stm32/g0 stm32/g4")
	set(OPENCM3_SRC_DIR  "${CMAKE_SOURCE_DIR}/lib/libopencm3")
	ExternalProject_Add(libopencm3
		SOURCE_DIR ${OPENCM3_SRC_DIR}
		CONFIGURE_COMMAND ""
		BUILD_COMMAND make -C ${OPENCM3_SRC_DIR} TARGETS=${OPENCM3_TARGETS} V=1
		INSTALL_COMMAND ""
		#INSTALL_COMMAND make install DESTDIR=${CMAKE_CURRENT_BINARY_DIR}/libopencm3
		#BUILD_IN_SOURCE true
		#BUILD_BYPRODUCTS ${OPENCM3_SRC_DIR}/lib/libopencm3_stm32f0.a ${OPENCM3_SRC_DIR}/lib/libopencm3_stm32g0.a ${OPENCM3_SRC_DIR}/lib/libopencm3_stm32g4.a
	)
	set(LIBOPENCM3_DIR ${OPENCM3_SRC_DIR})
endif(BUILD_DEPS)

if(UNIX AND NOT CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT) # Use 'CMAKE_INSTALL_PREFIX' as alternative system root
	set(CMAKE_PREFIX_PATH ${CMAKE_INSTALL_PREFIX})
	include(GNUInstallDirs)
	include_directories(${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_INCLUDEDIR})
	link_directories(${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR})
endif()
if(LIBOPENCM3_DIR)
	include_directories(${LIBOPENCM3_DIR}/include)
	link_directories(${LIBOPENCM3_DIR}/lib)
endif()
add_compile_options(-ffreestanding -ffunction-sections -fdata-sections -fsingle-precision-constant
	-Wall -Wextra -Wpedantic -Wundef -Wshadow -Wredundant-decls -Wstrict-prototypes -Wmissing-prototypes
	-Wno-variadic-macros -Wno-unused-result -Wno-unused-parameter -Wno-unused-label)
add_link_options(-nostartfiles -Wl,--gc-sections)

function(add_target name mcu)
	include(mcu/${mcu}/config.cmake)
	file(READ src/main.c main_c)
	string(REGEX MATCH "REVISION [0-9]+" rev ${main_c})
	string(REGEX REPLACE "[^0-9]+" "-rev" rev ${rev})
	set(elf ${name}${rev}.elf)
	set(bin ${name}${rev}.bin)
	set(hex ${name}${rev}.hex)
	file(GLOB srcs src/*.c mcu/${mcu}/*.c)
	add_executable(${elf} ${srcs})
if(BUILD_DEPS)
	add_dependencies(${elf} libopencm3)
endif(BUILD_DEPS)
	target_include_directories(${elf} PRIVATE src mcu/${mcu})
	target_compile_options(${elf} PRIVATE ${opts})
	target_compile_definitions(${elf} PRIVATE ${mcu} ${name} ${defs} TARGET_NAME="${name}" ${ARGN})
	target_link_options(${elf} PRIVATE -T${CMAKE_CURRENT_SOURCE_DIR}/mcu/${mcu}/config.ld -T${CMAKE_CURRENT_SOURCE_DIR}/mcu/common.ld ${opts})
	target_link_libraries(${elf} c_nano gcc nosys ${libs})
	add_custom_command(
		OUTPUT ${bin} ${hex}
		COMMAND arm-none-eabi-objcopy -O binary ${elf} ${bin}
		COMMAND arm-none-eabi-objcopy -O ihex ${elf} ${hex}
		DEPENDS ${elf}
	)
	add_custom_target(${name} ALL DEPENDS ${hex})
	add_custom_target(flash-${name} COMMAND st-flash --reset --connect-under-reset --format ihex write ${hex} DEPENDS ${hex})
endfunction()

add_subdirectory(boot)

add_target(AIKON1 STM32G071 DEAD_TIME=35 COMP_MAP=213 IO_PA6)
add_target(AIRBOT1 AT32F421 DEAD_TIME=66 COMP_MAP=321 SENS_MAP=0xA3A6 VOLT_MUL=738 CURR_MUL=30 LED_MAP=0xAFB3B4)
add_target(AIRBOT2 STM32F051 DEAD_TIME=26 COMP_MAP=321 SENS_MAP=0xA3 VOLT_MUL=738 IO_PA2)
add_target(DYS1 STM32F051 DEAD_TIME=26 COMP_MAP=123 LED_MAP=0xAFB3B4 LED_INV IO_PA2)
add_target(EMAX1 STM32F051 DEAD_TIME=26 COMP_MAP=123 IO_PA2)
add_target(ESCAPE1 STM32G071 DEAD_TIME=35 COMP_MAP=123 SENS_MAP=0xA5A4 VOLT_MUL=1100 CURR_MUL=30 LED_WS2812 IO_PA2 IO_AUX)
add_target(ESCAPE2 STM32G071 DEAD_TIME=35 COMP_MAP=123 SENS_MAP=0xA6 VOLT_MUL=1100 USE_BEC IO_PA2 IO_AUX)
add_target(ESCAPE3 AT32F421 DEAD_TIME=66 COMP_MAP=123 SENS_MAP=0xA6 VOLT_MUL=1100 USE_BEC IO_AUX)
add_target(FLYCOLOR1 STM32F051 DEAD_TIME=26 COMP_MAP=123 SENS_MAP=0xA6 VOLT_MUL=1100 LED_MAP=0xB5B4B3 IO_PA2)
add_target(FLYCOLOR2 STM32G071 DEAD_TIME=35 COMP_MAP=123 SENS_MAP=0xA6 VOLT_MUL=1100 LED_MAP=0xB8)
add_target(HAKRC1 STM32F051 DEAD_TIME=26 COMP_MAP=213 SENS_MAP=0xA3 VOLT_MUL=1100 LED_MAP=0xAFB5B3 LED_INV)
add_target(HAKRC2 AT32F421 DEAD_TIME=66 COMP_MAP=213 SENS_MAP=0xA3 VOLT_MUL=1100 LED_MAP=0xAFB5B3 LED_INV)
add_target(HGLRC1 STM32F051 DEAD_TIME=26 COMP_MAP=123 SENS_MAP=0xA6 VOLT_MUL=2100 IO_PA2)
add_target(IFLIGHT1 STM32F051 DEAD_TIME=26 COMP_MAP=321 SENS_MAP=0xA3A6 VOLT_MUL=1100 CURR_MUL=20 LED_MAP=0xB8B5B3 LED_INV)
add_target(IFLIGHT2 STM32G071 DEAD_TIME=35 COMP_MAP=213 SENS_MAP=0xA5A4 VOLT_MUL=1100 CURR_MUL=20 LED_WS2812 IO_PA6)
add_target(IFLIGHT3 STM32G071 DEAD_TIME=35 COMP_MAP=132 SENS_MAP=0xA0 VOLT_MUL=1100 LED_WS2812)
add_target(LUMENIER1 GD32F350 DEAD_TIME=57 COMP_MAP=321 SENS_MAP=0xA3 VOLT_MUL=1100)
add_target(NEUTRONRC1 STM32G071 DEAD_TIME=35 COMP_MAP=123 SENS_MAP=0xA6A4 VOLT_MUL=2100 CURR_MUL=50 LED_WS2812)
add_target(NEUTRONRC2 AT32F421 DEAD_TIME=66 COMP_MAP=321)
add_target(NEUTRONRC3 STM32G071 DEAD_TIME=35 COMP_MAP=123)
add_target(NEUTRONRC4 AT32F421 DEAD_TIME=66 COMP_MAP=321 INVERTED_HIGH)
add_target(SKYSTARS1 GD32E230 DEAD_TIME=40 COMP_MAP=321 SENS_MAP=0xA3 VOLT_MUL=1100 LED_MAP=0xB5B3AF)
add_target(TMOTOR1 STM32F051 DEAD_TIME=26 COMP_MAP=132 IO_PA2)
add_target(TMOTOR2 STM32F051 DEAD_TIME=26 COMP_MAP=321)
add_target(TMOTOR3 STM32G071 DEAD_TIME=35 COMP_MAP=231 IO_PA6)

# Experimental targets
add_target(_SPIN32 STSPIN32F0 DEAD_TIME=26 ANALOG) # Sensored drive using STEVAL_SPIN3201 board
add_target(_G431 STM32G431 DEAD_TIME=180 COMP_MAP=132 SENS_MAP=0xBFA6 VOLT_MUL=1600 CURR_MUL=100 USE_HSE=8 USE_PB1)
