# we have a fixed requirement for PCRE
set(PCRE_REQUIRED_MAJOR_VERSION 8)
set(PCRE_REQUIRED_MINOR_VERSION 41)
set(PCRE_REQUIRED_VERSION ${PCRE_REQUIRED_MAJOR_VERSION}.${PCRE_REQUIRED_MINOR_VERSION})

include (${CMAKE_MODULE_PATH}/pcre.cmake)
if (NOT CORRECT_PCRE_VERSION)
    message(STATUS "PCRE ${PCRE_REQUIRED_VERSION} not found, not building hscollider")
    return()
endif()

include_directories(${PCRE_INCLUDE_DIRS})

include(${CMAKE_MODULE_PATH}/backtrace.cmake)

# we need static libs - too much deep magic for shared libs
if (NOT BUILD_STATIC_LIBS)
    return ()
endif ()

CHECK_FUNCTION_EXISTS(sigaltstack HAVE_SIGALTSTACK)
CHECK_FUNCTION_EXISTS(sigaction HAVE_SIGACTION)
CHECK_FUNCTION_EXISTS(setrlimit HAVE_SETRLIMIT)

set_source_files_properties(
    ${CMAKE_CURRENT_BINARY_DIR}/ColliderCorporaParser.cpp
    PROPERTIES
    COMPILE_FLAGS "${RAGEL_C_FLAGS} -I${CMAKE_CURRENT_SOURCE_DIR}")

ragelmaker(ColliderCorporaParser.rl)

# only set these after all tests are done
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${EXTRA_C_FLAGS}")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${EXTRA_CXX_FLAGS}")

SET(hscollider_SOURCES
    common.h
    BoundedQueue.h
    Corpora.cpp
    FileCorpora.h
    FileCorpora.cpp
    ColliderCorporaParser.h
    ColliderCorporaParser.cpp
    NfaGeneratedCorpora.h
    NfaGeneratedCorpora.cpp
    GraphTruth.h
    GraphTruth.cpp
    GroundTruth.h
    GroundTruth.cpp
    UltimateTruth.h
    UltimateTruth.cpp
    ResultSet.h
    args.cpp
    args.h
    limit.cpp
    pcre_util.cpp
    sig.cpp
    sig.h
    DatabaseProxy.h
    Thread.h
    Thread.cpp
    main.cpp
)

set_source_files_properties(${hscollider_SOURCES} PROPERTIES
    INCLUDE_DIRECTORIES ${CMAKE_CURRENT_SOURCE_DIR})
add_executable(hscollider ${hscollider_SOURCES})
add_dependencies(hscollider ragel_ColliderCorporaParser)
add_dependencies(hscollider pcre)

if(NOT WIN32)
    target_link_libraries(hscollider hs ${PCRE_LDFLAGS} databaseutil
        expressionutil corpusomatic crosscompileutil pthread
    "${BACKTRACE_LDFLAGS}")

if(HAVE_BACKTRACE)
    set_source_files_properties(hscollider_SOURCES COMPILE_FLAGS
        "${BACKTRACE_CFLAGS}")
endif()
else() # WIN32
    target_link_libraries(hscollider hs ${PCRE_LDFLAGS} databaseutil
        expressionutil corpusomatic crosscompileutil)
endif()

add_custom_target(
    collide_quick_test
    COMMAND ${CMAKE_BINARY_DIR}/bin/hscollider
        -s ${CMAKE_SOURCE_DIR}/tools/hscollider/test_cases/signatures/collider_tests.txt
        -c ${CMAKE_SOURCE_DIR}/tools/hscollider/test_cases/corpora/*
        -Z0 -t3
    DEPENDS hscollider
)

add_custom_target(
    collide_quick_test_block
    COMMAND ${CMAKE_BINARY_DIR}/bin/hscollider
        -s ${CMAKE_SOURCE_DIR}/tools/hscollider/test_cases/signatures/collider_tests.txt
        -c ${CMAKE_SOURCE_DIR}/tools/hscollider/test_cases/corpora/*
        -Z0
    DEPENDS hscollider
)