add_library(
	util
	STATIC
	include/util/overloaded.h
	include/util/align.h
	include/util/bytes.h
	include/util/safe_time.h
	include/util/elf.h
	include/util/exit_code.h
	include/util/exec.h
	include/util/find.h
	include/util/float_to_half.h
	include/util/fs.h
	include/util/function_info.h
	include/util/instrset_detect.h
	include/util/lock_and_find.h
	include/util/log.h
	include/util/net_utils.h
	include/util/preprocessor.h
	include/util/pool.h
	include/util/string_utils.h
	include/util/system.h
	include/util/tracy.h
	include/util/types.h
	include/util/vector_utils.h
	src/util.cpp
	src/instrset_detect.cpp
)

target_include_directories(util PUBLIC include)
target_link_libraries(util PUBLIC ${Boost_LIBRARIES} config fmt spdlog http mem)
