# Copyright 2017-2018 by Martin Moene
#
# https://github.com/martinmoene/value-ptr-lite
#
# Distributed under the Boost Software License, Version 1.0.
# (See accompanying file LICENSE.txt or copy at http://www.boost.org/LICENSE_1_0.txt)


cmake_minimum_required( VERSION 3.5 )

project( test LANGUAGES CXX )

option( VALUE_PTR_LITE_COLOURISE_TEST "Colourise test output" OFF )

set( MAIN_BASE value_ptr-main ) # value_ptr-main
set( HDRNAME   value_ptr.hpp    )
set( SOURCES   ${MAIN_BASE}.t.cpp value_ptr.t.cpp )

set( HDRDIR  ${PROJECT_SOURCE_DIR}/../include/nonstd )
set( HDRPATH ${HDRDIR}/${HDRNAME} )

# Configure value_ptr-lite for testing:

set( OPTIONS "" )
set( VALUE_PTR_CONFIG )

if ( VALUE_PTR_LITE_COLOURISE_TEST )
    set( OPTIONS ${OPTIONS} -Dlest_FEATURE_COLOURISE=1 )
endif()

set( HAS_STD_FLAGS  FALSE )
set( HAS_CPP98_FLAG FALSE )
set( HAS_CPP11_FLAG FALSE )
set( HAS_CPP14_FLAG FALSE )
set( HAS_CPP17_FLAG FALSE )
set( HAS_CPP20_FLAG FALSE )
set( HAS_CPPLATEST_FLAG FALSE )

if( MSVC )
    message( STATUS "Matched: MSVC")

    set( HAS_STD_FLAGS TRUE )

    set( OPTIONS     -W3 -EHsc )
    set( DEFINITIONS -D_SCL_SECURE_NO_WARNINGS ${VALUE_PTR_CONFIG} )

    if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 19.00 )
        set( HAS_CPP14_FLAG TRUE )
        set( HAS_CPPLATEST_FLAG TRUE )
    endif()
    if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 19.11 )
        set( HAS_CPP17_FLAG TRUE )
    endif()

elseif( CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang|AppleClang" )
    message( STATUS "CompilerId: '${CMAKE_CXX_COMPILER_ID}'")

    set( HAS_STD_FLAGS  TRUE )
    set( HAS_CPP98_FLAG TRUE )

    set( OPTIONS     -Wall -Wextra -Wconversion -Wsign-conversion -Wno-missing-braces -fno-elide-constructors )
    set( DEFINITIONS ${VALUE_PTR_CONFIG} )

    # GNU: available -std flags depends on version
    if( CMAKE_CXX_COMPILER_ID MATCHES "GNU" )
        message( STATUS "Matched: GNU")

        if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 4.8.0 )
            set( HAS_CPP11_FLAG TRUE )
        endif()
        if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 4.9.2 )
            set( HAS_CPP14_FLAG TRUE )
        endif()
        if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 7.1.0 )
            set( HAS_CPP17_FLAG TRUE )
        endif()

    # AppleClang: available -std flags depends on version
    elseif( CMAKE_CXX_COMPILER_ID MATCHES "AppleClang" )
        message( STATUS "Matched: AppleClang")

        if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 5.0.0 )
            set( HAS_CPP11_FLAG TRUE )
        endif()
        if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 5.1.0 )
            set( HAS_CPP14_FLAG TRUE )
        endif()
        if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 9.2.0 )
            set( HAS_CPP17_FLAG TRUE )
        endif()

    # Clang: available -std flags depends on version
    elseif( CMAKE_CXX_COMPILER_ID MATCHES "Clang" )
        message( STATUS "Matched: Clang")

        if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 3.3.0 )
            set( HAS_CPP11_FLAG TRUE )
        endif()
        if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 3.4.0 )
            set( HAS_CPP14_FLAG TRUE )
        endif()
        if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 5.0.0 )
            set( HAS_CPP17_FLAG TRUE )
        endif()
    endif()

elseif( CMAKE_CXX_COMPILER_ID MATCHES "Intel" )
    # as is
    message( STATUS "Matched: Intel")
else()
    # as is
    message( STATUS "Matched: nothing")
endif()

# enable MS C++ Core Guidelines checker if MSVC:

function( enable_msvs_guideline_checker target )
    if( MSVC )
        set_target_properties( ${target} PROPERTIES
        VS_GLOBAL_EnableCppCoreCheck true
        VS_GLOBAL_CodeAnalysisRuleSet CppCoreCheckRules.ruleset
        VS_GLOBAL_RunCodeAnalysis true )
    endif()
endfunction()

# make target, compile for given standard if specified:

function( make_target target std )
    message( STATUS "Make target: '${std}'" )

    add_executable            ( ${target} ${SOURCES} ${HDRPATH} )
    target_compile_options    ( ${target} PRIVATE ${OPTIONS} )
    target_compile_definitions( ${target} PRIVATE ${DEFINITIONS} )
    target_include_directories( ${target} PRIVATE ${HDRDIR} )
    if( std )
        if( MSVC )
            target_compile_options( ${target} PRIVATE -std:c++${std} )
        else()
            # Necessary for clang 3.x:
            target_compile_options( ${target} PRIVATE -std=c++${std} )
            # Ok for clang 4 and later:
            # set( CMAKE_CXX_STANDARD ${std} )
            # set( CMAKE_CXX_STANDARD_REQUIRED ON )
            # set( CMAKE_CXX_EXTENSIONS OFF )
        endif()
    endif()
endfunction()

# add generic executable, unless -std flags can be specified:

if( NOT HAS_STD_FLAGS )
    make_target( ${MAIN_BASE}.t "" )
else()
    # unconditionally add C++98 variant as MSVC has no option for it:
    if( HAS_CPP98_FLAG )
        make_target( ${MAIN_BASE}-cpp98.t 98 )
    else()
        make_target( ${MAIN_BASE}-cpp98.t "" )
    endif()

    if( HAS_CPP11_FLAG )
        make_target( ${MAIN_BASE}-cpp11.t 11 )
    endif()

    if( HAS_CPP14_FLAG )
        make_target( ${MAIN_BASE}-cpp14.t 14 )
    endif()

    if( HAS_CPP17_FLAG )
        set( std17 17 )
        if( CMAKE_CXX_COMPILER_ID MATCHES "AppleClang" )
            set( std17 1z )
        endif()
        make_target( ${MAIN_BASE}-cpp17.t ${std17} )
        enable_msvs_guideline_checker( ${MAIN_BASE}-cpp17.t )
    endif()

    if( HAS_CPPLATEST_FLAG )
        make_target( ${MAIN_BASE}-cpplatest.t latest )
    endif()
endif()

# configure unit tests via CTest:

enable_testing()

if( HAS_STD_FLAGS )
    # unconditionally add C++98 variant for MSVC:
    add_test(     NAME test-cpp98     COMMAND ${MAIN_BASE}-cpp98.t )

    if( HAS_CPP11_FLAG )
        add_test( NAME test-cpp11     COMMAND ${MAIN_BASE}-cpp11.t )
    endif()
    if( HAS_CPP14_FLAG )
        add_test( NAME test-cpp14     COMMAND ${MAIN_BASE}-cpp14.t )
    endif()
    if( HAS_CPP17_FLAG )
        add_test( NAME test-cpp17     COMMAND ${MAIN_BASE}-cpp17.t )
    endif()
    if( HAS_CPPLATEST_FLAG )
        add_test( NAME test-cpplatest COMMAND ${MAIN_BASE}-cpplatest.t )
    endif()
else()
    add_test(     NAME test           COMMAND ${MAIN_BASE}.t --pass )
    add_test(     NAME list_version   COMMAND ${MAIN_BASE}.t --version )
    add_test(     NAME list_tags      COMMAND ${MAIN_BASE}.t --list-tags )
    add_test(     NAME list_tests     COMMAND ${MAIN_BASE}.t --list-tests )
endif()

# end of file
