# PROJECT: termite
cmake_minimum_required(VERSION 3.3)

# Source Files
file(GLOB SOURCE_FILES RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} "*.c*" "*.h")
file(GLOB INCLUDE_FILES RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} "../../include/termite/*.h")
if (IOS)
    list(APPEND SOURCE_FILES "ios_window.mm")
endif()
source_group(source FILES ${SOURCE_FILES})
source_group(include FILES ${INCLUDE_FILES})

## Shaders
set(SHADER_SOURCES "shaders/ddraw.vsc"          "shaders/ddraw.fsc" 
                   "shaders/vg.vsc"             "shaders/vg.fsc"
                   "shaders/blit.vsc"           "shaders/blit.fsc"
                   "shaders/font_normal.vsc"    "shaders/font_normal.fsc"
                   "shaders/vignette_sepia.vsc" "shaders/vignette_sepia.fsc")

source_group(shaders FILES ${SHADER_SOURCES})
bgfx_add_shaders("${SHADER_SOURCES}" IGNORE "../shaders" "shaders_h" TRUE IGNORE SHADER_GEN_FILES)
source_group(shaders\\compiled FILES ${SHADER_GEN_FILES})

# Sprite shaders
set(SPRITE_SHADER_SOURCES "shaders/sprite.vsc" "shaders/sprite.fsc")
source_group(shaders FILES ${SPRITE_SHADER_SOURCES})
bgfx_add_shaders("${SPRITE_SHADER_SOURCES}" IGNORE "../shaders" "shaders_h" TRUE IGNORE SPRITE_NORMAL_FILES) 
source_group(shaders\\compiled FILES ${SPRITE_NORMAL_FILES})
bgfx_add_shaders("${SPRITE_SHADER_SOURCES}" "COLOR_ADD=1" "../shaders" "shaders_h" TRUE "add" SPRITE_COLOR_ADD_FILES) 
source_group(shaders\\compiled FILES ${SPRITE_COLOR_ADD_FILES})

# Fade shaders
set(FADE_SHADER_SOURCES "shaders/effect_fade.vsc" "shaders/effect_fade.fsc")
source_group(shaders FILES ${FADE_SHADER_SOURCES})
bgfx_add_shaders("${FADE_SHADER_SOURCES}" "FADE_OUT;FADE_COLOR" "../shaders" "shaders_h" TRUE "out_color" FADEOUT_COLOR_FILES) 
source_group(shaders\\compiled FILES ${FADEOUT_COLOR_FILES})
bgfx_add_shaders("${FADE_SHADER_SOURCES}" "FADE_IN;FADE_COLOR" "../shaders" "shaders_h" TRUE "in_color" FADEIN_COLOR_FILES) 
source_group(shaders\\compiled FILES ${FADEIN_COLOR_FILES})
bgfx_add_shaders("${FADE_SHADER_SOURCES}" "FADE_OUT;FADE_ALPHA" "../shaders" "shaders_h" TRUE "out_alpha" FADEOUT_ALPHA_FILES) 
source_group(shaders\\compiled FILES ${FADEOUT_ALPHA_FILES})
bgfx_add_shaders("${FADE_SHADER_SOURCES}" "FADE_IN;FADE_ALPHA" "../shaders" "shaders_h" TRUE "in_alpha" FADEIN_ALPHA_FILES) 
source_group(shaders\\compiled FILES ${FADEIN_ALPHA_FILES})

# Font shaders
set(FONT_SHADER_SOURCES "shaders/font_df.vsc" "shaders/font_df.fsc")
source_group(shaders FILES ${FONT_SHADER_SOURCES})
bgfx_add_shaders("${FONT_SHADER_SOURCES}" IGNORE "../shaders" "shaders_h" TRUE IGNORE FONT_DF_FILES)
source_group(shaders\\compiled FILES ${FONT_DF_FILES})
bgfx_add_shaders("${FONT_SHADER_SOURCES}" "SHADOW" "../shaders" "shaders_h" TRUE "shadow" FONT_DF_SHADOW_FILES)
source_group(shaders\\compiled FILES ${FONT_DF_SHADOW_FILES})
bgfx_add_shaders("${FONT_SHADER_SOURCES}" "OUTLINE" "../shaders" "shaders_h" TRUE "outline" FONT_DF_OUTLINE_FILES)
source_group(shaders\\compiled FILES ${FONT_DF_OUTLINE_FILES})

# Blur Shaders
# Blur shader has a constant parameter BLUR_KERNEL_SIZE which should be odd number a
set(BLUR_KERNEL_SIZE "9" CACHE STRING "Blur Kernel size (Odd Number)")
add_definitions(-DBLUR_KERNEL_SIZE=${BLUR_KERNEL_SIZE})

set(BLUR_SHADER_SOURCES "shaders/blur.vsc" "shaders/blur.fsc")
source_group(shaders FILES ${BLUR_SHADER_SOURCES})
bgfx_add_shaders("${BLUR_SHADER_SOURCES}" "KERNEL_SIZE=${BLUR_KERNEL_SIZE}" "../shaders" "shaders_h" TRUE IGNORE BLUR_SHADER_FILES)

add_library(termite ${BUILD_LIBRARY_TYPE} 
            ${SOURCE_FILES} 
            ${INCLUDE_FILES} 
            ${SHADER_SOURCES} 
            ${SPRITE_SHADER_SOURCES}
            ${FADE_SHADER_SOURCES} 
            ${BLUR_SHADER_SOURCES}
            ${SHADER_GEN_FILES} 
            ${SPRITE_NORMAL_FILES}
            ${SPRITE_COLOR_ADD_FILES}
            ${FADEOUT_COLOR_FILES} 
            ${FADEIN_COLOR_FILES} 
            ${FADEOUT_ALPHA_FILES} 
            ${FADEIN_ALPHA_FILES}
            ${FONT_DF_FILES} 
            ${FONT_DF_SHADOW_FILES} 
            ${FONT_DF_OUTLINE_FILES} 
            ${BLUR_SHADER_FILES})

# STD_LIBS
# libraries for unix/posix
if (ANDROID)
    set(STD_LIBS log dl GLESv2 EGL android)
elseif (UNIX)
    set(STD_LIBS pthread dl)
endif()

# PRIVATE_LIBS
if (CMAKE_COMPILER_IS_GNUCC)
    if (${CMAKE_CXX_COMPILER_VERSION} LESS "6.0.0" AND ${CMAKE_SYSTEM_NAME} MATCHES "Linux")
        message(WARNING "Using -lgcc -lgcc_s workaround for gcc bug")
        set(PRIVATE_LIBS gcc gcc_s)        
    endif()
endif()

# statically linked plugin libraries
if (BUILD_STATIC)
    if (ANDROID)
        set(DISK_DRIVER android_assets_driver)
    elseif (USE_DISK_LITE_DRIVER)
        set(DISK_DRIVER disk_lite_driver)
    else()
        set(DISK_DRIVER disk_driver)
    endif()
    set(PLUGIN_LIBS bgfx_driver box2d_driver ${DISK_DRIVER})

    if (USE_SDL2_MIXER)
        set(PLUGIN_LIBS ${PLUGIN_LIBS} sdl_mixer_driver)
    endif()
endif()

if (NOT IOS)
    set(FCONTEXT_LIB fcontext)
else()
    add_dependencies(termite fcontext)
    get_filename_component(FCONTEXT_LIB ${CMAKE_CURRENT_SOURCE_DIR}/../../out/libfcontext.a ABSOLUTE)
    get_filename_component(FCONTEXT_INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/../../deps/deboost.context/include ABSOLUTE)
endif()

target_link_libraries(termite 
    PRIVATE ${FCONTEXT_LIB} imgui imgui_impl remotery ${PLUGIN_LIBS} ${PRIVATE_LIBS}
    PUBLIC bx rapidjson AHEasing ${STD_LIBS})

# 
get_filename_component(TERMITE_INCLUDE_DIR ../../include/termite ABSOLUTE)
target_include_directories(termite PUBLIC ${TERMITE_INCLUDE_DIR} ${DEP_ROOT_DIR} ${FCONTEXT_INCLUDE_DIR})
if (USE_SDL2)
    target_include_directories(termite PUBLIC ${SDL2_INCLUDE_DIR})
    target_link_libraries(termite PUBLIC ${SDL2_LIBRARY})
endif()

# PCH
if (NOT ANDROID)
    add_precompiled_header(termite pch.h)
endif()

## Install
install(TARGETS termite
        RUNTIME DESTINATION bin
        ARCHIVE DESTINATION lib
        LIBRARY DESTINATION lib)
install(FILES ${INCLUDE_FILES} DESTINATION include/termite)

