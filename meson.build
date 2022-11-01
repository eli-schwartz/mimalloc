project('mimalloc', 'c', 'cpp',
    version: '1.7.6',
    meson_version: '>=0.53.0',
    default_options: ['c_std=c11', 'cpp_std=c++17', 'warning_level=3'],
)

cc = meson.get_compiler('c')
cc_id = cc.get_id()

sources = files(
    'src/stats.c',
    'src/random.c',
    'src/os.c',
    'src/bitmap.c',
    'src/arena.c',
    'src/region.c',
    'src/segment.c',
    'src/page.c',
    'src/alloc.c',
    'src/alloc-aligned.c',
    'src/alloc-posix.c',
    'src/heap.c',
    'src/options.c',
    'src/init.c',
)

defines = []

if get_option('override')
    if host_machine.system() == 'darwin'
        if get_option('osx_zone')
            sources += files('src/alloc-override-osx.c')
            defines += '-DMI_OSX_ZONE=1'
        endif
        if get_option('osx_interpose')
            defines += '-DMI_OSX_INTERPOSE=1'
        endif
    endif
endif

if host_machine.system() == 'windows'
    if not get_option('win_redirect') or host_machine.cpu_family() == 'arm'
        # define is negative for backward compatibility
        defines += '-DMI_WIN_NOREDIRECT=1'
    endif
endif

if get_option('secure')
    defines += '-DMI_SECURE=4'
endif


valgrind = dependency('valgrind', required: get_option('valgrind'))
if valgrind.found()
    # cmake checks for headers in unreliable subdir
    defines += '-DMI_VALGRIND=1'
endif

if get_option('see_asm')
    add_project_arguments('-save-temps', language: 'c')
endif

if get_option('debug_full')
    defines += '-DMI_DEBUG=3'
endif
if not get_option('padding')
    defines += '-DMI_PADDING=0'
endif

foreach opt: ['skip_collect_on_exit', 'xmalloc', 'show_errors']
    if get_option(opt)
        defines += '-DMI_@0@=1'.format(opt.to_upper())
    endif
endforeach

add_project_arguments(defines, language: 'c')

if get_option('b_sanitize').contains('undefined')
    add_project_arguments('-fno-sanitize-recover=undefined', language: 'c')
    warning('should be compiled with C++ to use sanitizer?')
endif

add_project_arguments(cc.get_supported_arguments('-Wno-unknown-pragmas', '-Wno-static-in-inline'), language: 'c')

# should this check family instead?
if (cc_id.contains('clang') or cc_id.contains('intel') or cc_id == 'gcc') and host_machine.system() != 'haiku'
    if get_option('local_dynamic_tls')
        add_project_arguments('-ftls-model=local-dynamic', language: 'c')
    else
        add_project_arguments('-ftls-model=initial-exec', language: 'c')
    endif
    if get_option('override')
        add_project_arguments('-fno-builtin-malloc', language: 'c')
    endif
endif


if host_machine.system() == 'windows'
    deps = [
        cc.find_library('psapi'),
        cc.find_library('shell32'),
        cc.find_library('user32'),
        cc.find_library('advapi32'),
        cc.find_library('bcrypt'),
    ]
    if get_option('redirect')
        name = 'mimalloc-redirect'
        if host_machine.cpu_family() == 'x86'
            name += '32'
        endif
        deps += cc.find_library(name, dirs: 'bin')
    endif
else
    deps = [
        dependency('threads', required: false),
        cc.find_library('rt', required: false),
        cc.find_library('atomic', required: false),
    ]
endif


basename = 'mimalloc'
if get_option('secure')
    basename += '-secure'
endif
if valgrind.found()
    basename += '-valgrind'
endif

libflags = []
# It is only possible to override malloc on Windows when building as a DLL.
if host_machine.system() != 'windows' or get_option('default_library') == 'shared'
    libflags += '-DMI_MALLOC_OVERRIDE'
endif


if get_option('use_cxx')
    newsources = []
    fs = import('fs')
    foreach x: sources
        if meson.version().version_compare('>=0.64.0')
            newsources += fs.copyfile(x, fs.stem(x)+'.cpp')
        else
            newsources += configure_file(input: x, output: fs.stem(x)+'.cpp', copy: true)
        endif
    endforeach
    sources = newsources
endif

basic = declare_dependency(dependencies: deps, include_directories: 'include')
libdep = declare_dependency(sources: sources, compile_args: libflags, dependencies: basic, include_directories: 'src')

# shared/static libraries need unique c_args, so using library() doesn't work
if get_option('default_library') != 'shared'
    mimalloc_static = static_library(
        basename,
        dependencies: libdep,
        gnu_symbol_visibility: 'hidden',
        install: true,
    )
    mimalloc_lib = mimalloc_static
    mimalloc_static_dep = declare_dependency(link_with: mimalloc_static, dependencies: basic)
endif
if get_option('default_library') != 'static'
    mimalloc_shared = shared_library(
        basename,
        c_args: ['-DMI_SHARED_LIB', '-DMI_SHARED_LIB_EXPORT'],
        cpp_args: ['-DMI_SHARED_LIB', '-DMI_SHARED_LIB_EXPORT'],
        version: meson.project_version(),
        dependencies: libdep,
        gnu_symbol_visibility: 'hidden',
        install: true,
    )
    mimalloc_lib = mimalloc_shared
endif

import('pkgconfig').generate(
    mimalloc_lib,
    filebase: 'mimalloc',
    description: 'A compact general purpose allocator with excellent performance',
)

mimalloc_dep = declare_dependency(link_with: mimalloc_lib, dependencies: basic)
if meson.version().version_compare('>=0.54.0')
    meson.override_dependency('mimalloc', mimalloc_dep)
endif

install_headers('include/mimalloc.h', 'include/mimalloc-override.h', 'include/mimalloc-new-delete.h')


# install cmake stuff?

subdir('test')
