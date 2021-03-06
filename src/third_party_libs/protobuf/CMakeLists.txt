set(CMAKE_DISABLE_SOURCE_CHANGES ON)
set(CMAKE_DISABLE_IN_SOURCE_BUILD ON)
# protoc segfaults if compiled with ubsan
set(NO_UBSAN TRUE)
# ned 2014-03-21: The above line works for protobuf, but has no effect on protoc which
#                 still gains the -fsanitize=undefined. I've wasted enough time on this
#                 so I'm pressing ahead with the below hack
if(HAVE_UNDEFINED_BEHAVIOR_SANITIZER)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-sanitize=undefined")
endif()

project(Protobuf)

file(GLOB_RECURSE LIB_PROTBUF_SOURCES "${PROJECT_SOURCE_DIR}/src/google/protobuf/*.cc")
file(GLOB_RECURSE LIB_PROTBUF_HEADERS "${PROJECT_SOURCE_DIR}/src/google/protobuf/*.h")
set(PROTOBUF_DIR ${LIB_PROTBUF_SOURCES} ${LIB_PROTBUF_HEADERS})

file(GLOB_RECURSE LIB_PROTOC_SOURCES "${PROJECT_SOURCE_DIR}/src/google/protobuf/compiler/*.cc")
file(GLOB_RECURSE LIB_PROTOC_HEADERS "${PROJECT_SOURCE_DIR}/src/google/protobuf/compiler/*.h")
set(PROTOC_DIR ${LIB_PROTOC_SOURCES} ${LIB_PROTOC_HEADERS})

file(GLOB_RECURSE PROTOC_TESTS_SOURCES "${PROJECT_SOURCE_DIR}/src/google/protobuf/tests/*.cc")
file(GLOB_RECURSE PROTOC_TESTS_HEADERS "${PROJECT_SOURCE_DIR}/src/google/protobuf/tests/*.h")
set(TESTS ${PROTOC_TESTS_SOURCES} ${PROTOC_TESTS_HEADERS})

list(REMOVE_ITEM PROTOBUF_DIR ${PROTOC_DIR} ${TESTS})

add_library(protobuf ${PROTOBUF_DIR})
ms_target_include_system_dirs(protobuf PUBLIC ${PROJECT_SOURCE_DIR}/src)

add_executable(protoc ${PROTOC_DIR})
set_target_properties(protobuf protoc PROPERTIES FOLDER "Third Party/Google Protocol Buffers")

set(AllStaticLibsForCurrentProject protobuf)
set(AllExesForCurrentProject protoc)
include(../../../cmake_modules/standard_flags.cmake)

target_compile_definitions(protobuf
  PUBLIC
    $<$<AND:$<BOOL:${JUST_THREAD_DEADLOCK_CHECK}>,$<CONFIG:Debug>>:_JUST_THREAD_DEADLOCK_CHECK>
    $<$<BOOL:${UNIX}>:HAVE_PTHREAD>
)

target_compile_options(protobuf
  PUBLIC
    $<$<BOOL:${MSVC}>:
        $<$<CONFIG:Release>:/O2 /GL /MD>
        $<$<CONFIG:Debug>:/Od /MDd /Zi>
        $<$<CONFIG:MinSizeRel>:/MD>
        $<$<CONFIG:RelWithDebInfo>:/O2 /GL /MD /Zi>
    >
    $<$<BOOL:${UNIX}>:-std=c++11 ${LibCXX}>
  PRIVATE
    $<$<BOOL:${MSVC}>:/W3 /wd4018 /wd4244 /wd4267 /wd4800 /wd4996>
    $<$<BOOL:${UNIX}>:-w>
)

target_compile_definitions(protoc
  PUBLIC
    $<$<AND:$<BOOL:${JUST_THREAD_DEADLOCK_CHECK}>,$<CONFIG:Debug>>:_JUST_THREAD_DEADLOCK_CHECK>
)

target_compile_options(protoc
  PRIVATE
    $<$<BOOL:${MSVC}>:/W3 /wd4018 /wd4244 /wd4267 /wd4800 /wd4996>
    $<$<BOOL:${UNIX}>:-w>
)

target_link_libraries(protobuf ${JustThread_LIBRARIES})
if(TARGET check_sanitizer_blacklist)
  add_dependencies(protobuf check_sanitizer_blacklist)
endif()
target_link_libraries(protoc protobuf)
if(UNIX)
  target_link_libraries(protoc -lpthread)
endif()
