# SPDX-License-Identifier: MIT

if(CONFIG_ZMK_SPLIT_BLE_ANY_CENTRAL)
  zephyr_library()
  zephyr_library_include_directories(${APPLICATION_SOURCE_DIR}/include)
  zephyr_library_sources(src/split_ble_any_central.c)
endif()
