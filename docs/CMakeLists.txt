# Copyright (C) 2018-2020 Intel Corporation
# SPDX-License-Identifier: Apache-2.0
#

cmake_minimum_required(VERSION 2.8)

find_package(Doxygen REQUIRED dot)
find_package(Python3 COMPONENTS Interpreter)
find_package(LATEX)

if(NOT DOXYGEN_FOUND)
    message(FATAL_ERROR "Doxygen is required to build the documentation")
endif()

if(NOT Python3_FOUND)
    message(FATAL_ERROR "Python3 is required to build the documentation")
endif()

if(NOT LATEX_FOUND)
    message(FATAL_ERROR "LATEX is required to build the documentation")
endif()

set(DOXYGEN_DIR ${CMAKE_CURRENT_SOURCE_DIR}/doxygen)
set(IE_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/../inference-engine)
set(PYTHON_API_IN ${IE_SOURCE_DIR}/ie_bridges/python/src/openvino/inference_engine/ie_api.pyx)
set(C_API ${IE_SOURCE_DIR}/ie_bridges/c/include)
set(DOCS_DIR ${CMAKE_CURRENT_BINARY_DIR}/docs)
set(PYTHON_API_OUT ${DOCS_DIR}/python_api/ie_api.pyx)
set(PLUGIN_API_DIR ${DOCS_DIR}/IE_PLUGIN_DG)

if(NOT ENABLE_DOCKER)
    add_subdirectory(examples)

    # Detect nGraph
    find_package(ngraph QUIET)
    if(NOT ngraph_FOUND)
        set(ngraph_DIR ${CMAKE_BINARY_DIR}/ngraph)
    endif()

    # Detect InferenceEngine
    find_package(InferenceEngine QUIET)
    if(NOT InferenceEngine_FOUND)
        set(InferenceEngine_DIR ${CMAKE_BINARY_DIR})
    endif()

    add_subdirectory(template_extension)

    set(all_docs_targets
        ie_docs_examples
        template_extension
        templatePlugin TemplateBehaviorTests TemplateFunctionalTests)
    foreach(target_name IN LISTS all_docs_targets)
        if (TARGET ${target_name})
            set_target_properties(${target_name} PROPERTIES FOLDER docs)
        endif()
    endforeach()
endif()

# OpenVINO docs

file(COPY "${CMAKE_CURRENT_SOURCE_DIR}"
     DESTINATION "${DOCS_DIR}"
     FILES_MATCHING PATTERN *.md PATTERN *.png PATTERN *.gif
)

file(COPY "${IE_SOURCE_DIR}"
     DESTINATION "${DOCS_DIR}"
     FILES_MATCHING PATTERN *.md PATTERN *.png PATTERN *.gif
)

configure_file("${PYTHON_API_IN}" "${PYTHON_API_OUT}" @ONLY)

set(IE_CONFIG ${DOCS_DIR}/ie_docs)
set(C_CONFIG ${DOCS_DIR}/ie_c_api)
set(PY_CONFIG ${DOCS_DIR}/ie_py_api)
set(PLUGIN_CONFIG ${DOCS_DIR}/ie_plugin_api)
set(IE_LAYOUT ${DOCS_DIR}/ie_docs.xml)
set(C_LAYOUT ${DOCS_DIR}/ie_c_api.xml)
set(PY_LAYOUT ${DOCS_DIR}/ie_py_api.xml)
set(PLUGIN_LAYOUT ${DOCS_DIR}/ie_plugin_api.xml)

configure_file(${DOXYGEN_DIR}/ie_docs ${IE_CONFIG} @ONLY)
configure_file(${DOXYGEN_DIR}/ie_c_api ${C_CONFIG} @ONLY)
configure_file(${DOXYGEN_DIR}/ie_py_api ${PY_CONFIG} @ONLY)
configure_file(${DOXYGEN_DIR}/ie_plugin_api ${PLUGIN_CONFIG} @ONLY)
configure_file(${DOXYGEN_DIR}/ie_docs.xml ${IE_LAYOUT} @ONLY)
configure_file(${DOXYGEN_DIR}/ie_c_api.xml ${C_LAYOUT} @ONLY)
configure_file(${DOXYGEN_DIR}/ie_py_api.xml ${PY_LAYOUT} @ONLY)
configure_file(${DOXYGEN_DIR}/ie_plugin_api.xml ${PLUGIN_LAYOUT} @ONLY)

# Preprocessing scripts
set(DOXY_MD_FILTER ${DOXYGEN_DIR}/doxy_md_filter.py)
set(PYX_FILTER ${DOXYGEN_DIR}/pyx_filter.py)

# Execute preprocessing scripts
execute_process(COMMAND ${Python3_EXECUTABLE} ${DOXY_MD_FILTER} ${DOCS_DIR})
execute_process(COMMAND ${Python3_EXECUTABLE} ${PYX_FILTER} ${PYTHON_API_OUT})

add_custom_target(c_api
                  COMMAND ${DOXYGEN_EXECUTABLE} ${C_CONFIG}
                  WORKING_DIRECTORY ${DOCS_DIR}
                  COMMENT "Generating C API Reference"
                  VERBATIM
)

add_custom_target(py_api
                  COMMAND ${DOXYGEN_EXECUTABLE} ${PY_CONFIG}
                  WORKING_DIRECTORY ${DOCS_DIR}
                  COMMENT "Generating Python API Reference"
                  VERBATIM
)

add_custom_target(plugin_api
                  COMMAND ${DOXYGEN_EXECUTABLE} ${PLUGIN_CONFIG}
                  WORKING_DIRECTORY ${DOCS_DIR}
                  COMMENT "Generating Plugin API Reference"
                  VERBATIM
)

add_custom_target(ie_docs
                  COMMAND ${DOXYGEN_EXECUTABLE} ${IE_CONFIG}
                  WORKING_DIRECTORY ${DOCS_DIR}
                  VERBATIM
)

add_custom_target(openvino_docs
                  DEPENDS c_api 
                  DEPENDS py_api
                  DEPENDS ie_docs
                  DEPENDS plugin_api
                  COMMENT "Generating OpenVINO documentation"
                  VERBATIM
)

set_target_properties(ie_docs PROPERTIES FOLDER docs)

find_program(browser NAMES xdg-open)
if(browser)
    add_custom_target(ie_docs_open
                      COMMAND ${browser} "${OpenVINO_MAIN_SOURCE_DIR}/doc/html/index.html"
                      DEPENDS ie_docs
                      COMMENT "Open OpenVINO documentation"
                      VERBATIM)
    set_target_properties(ie_docs_open PROPERTIES FOLDER docs)
endif()
