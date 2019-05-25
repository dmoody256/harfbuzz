import os
import sys
import glob
import re
import time
import datetime
import atexit
import platform
import fileinput
import subprocess

start_time = time.time()

import SCons.Action
import SCons.Script.Main

from BuildUtils.SconsUtils import SetBuildJobs, SetupBuildEnv, ProgressCounter, display_build_status
from BuildUtils.ColorPrinter import ColorPrinter
from BuildUtils.FindPackages import FindFreetype, FindGraphite2, FindGlib, FindIcu, FindCairo
from BuildUtils.ConfigureChecks import *


def CreateNewEnv():
    
    p = ColorPrinter()
    SetupOptions()

    harfbuzz_options = [
        'option_have_freetype',
        'option_have_graphite2',
        'option_have_glib',
        'option_have_icu',
        'option_have_gobject',
        'option_have_introspection',
        'option_build_subset',
        'option_build_tests',
        'option_build_shared',
        'option_build_utils',
    ]

    if sys.platform == 'win32':
        harfbuzz_options += [
            'option_have_uniscribe',
            'option_have_directwrite',
        ]
           
    if sys.platform == 'darwin':
        harfbuzz_options += [
            'option_have_coretext',
        ]

    env = Environment(
        DEBUG_BUILD=GetOption('option_debug'),
        VERBOSE_COMPILE=GetOption('option_verbose'),
        RECONF=GetOption('option_reconfigure'),
        HARFBUZZ_CHECK=GetOption('option_check')
    )

    for option in harfbuzz_options:
        var_name = option.replace('option_', '').upper()
        env[var_name] = GetOption(option)

    if env['BUILD_UTILS']:
        env['HAVE_GLIB'] = True
        env['HAVE_FREETYPE'] = True

    if env['HAVE_GOBJECT']:
        env['HAVE_GLIB'] = True

    if env['HAVE_INTROSPECTION']:
        env['HAVE_GOBJECT'] = True
        env['HAVE_GLIB'] = True

    if env['HARFBUZZ_CHECK']:
        env['BUILD_SHARED'] = True
        env['BUILD_UTILS'] = True
        env['HAVE_ICU'] = True
        env['HAVE_GLIB'] = True
        env['HAVE_GOBJECT'] = True
        env['HAVE_INTROSPECTION'] = True
        env['HAVE_FREETYPE'] = True
        env['HAVE_GRAPHITE2'] = True
        if sys.platform == 'win32':
            env['HAVE_UNISCRIBE'] = True
            env['HAVE_DIRECTWRITE'] = True
        if sys.platform == 'darwin':
            env['HAVE_CORETEXT'] = True

    env['PROJECT_DIR'] = os.path.abspath(Dir('.').abspath).replace('\\', '/')
    env['BUILD_DIR'] = 'build'
    env.Execute(Mkdir('build'))

    SetBuildJobs(env)
    GetHarfbuzzVersion(env)
    progress = ProgressCounter()

    project_sources = []
    project_headers = []
    gobject_sources = []
    gobject_headers = []
    gobject_gen_sources = []
    gobject_gen_headers = []
    gobject_structs_headers = []
    project_extra_sources = []
    project_extra_headers = []
    subset_project_sources = []
    subset_project_headers = []
    

    sources = (
        GetSources('HB_BASE_sources', 'repo/src/Makefile.sources') +
        GetSources('HB_BASE_RAGEL_GENERATED_sources', 'repo/src/Makefile.sources') +
        GetSources('HB_FALLBACK_sources', 'repo/src/Makefile.sources') +
        GetSources('HB_BASE_headers', 'repo/src/Makefile.sources'))

    SortSources(
        project_sources, 
        project_headers, 
        sources
    )

    SortSources(
        subset_project_sources, 
        subset_project_headers, 
        GetSources('HB_SUBSET_sources', 'repo/src/Makefile.sources')
    )
    
    if env['HAVE_FREETYPE']:
        if not FindFreetype(env, conf_dir=env['BUILD_DIR']):
            p.ErrorPrint(
                "Requested build with Freetype, but couldn't find it.")
            return
        env.Append(CPPDEFINES=[('HAVE_FREETYPE', '1')])
        project_sources += ['repo/src/hb-ft.cc']
        project_headers += ['repo/src/hb-ft.h']

    if env['HAVE_GRAPHITE2'] and FindGraphite2(env, conf_dir=env['BUILD_DIR']):
        env.Append(CPPDEFINES=['HAVE_GRAPHITE2'])
        project_sources += ['repo/src/hb-graphite2.cc']
        project_headers += ['repo/src/hb-graphite2.h']
    
    if env['HAVE_GLIB'] and FindGlib(env, conf_dir=env['BUILD_DIR']):
        env.Append(CPPDEFINES=['HAVE_GLIB'])
        project_sources += ['repo/src/hb-glib.cc']
        project_headers += ['repo/src/hb-glib.h']

    if env['HAVE_ICU'] and FindIcu(env, conf_dir=env['BUILD_DIR']):
        env.Append(CPPDEFINES=['HAVE_ICU'])
        project_sources += ['repo/src/hb-icu.cc']
        project_headers += ['repo/src/hb-icu.h']

    if sys.platform == 'darwin':
        if env['HAVE_CORETEXT']:
            env.Append(CPPDEFINES=['HAVE_CORETEXT'])
            project_sources += ['repo/src/hb-coretext.cc']
            project_headers += ['repo/src/hb-coretext.h']

    if sys.platform == 'win32':
        if env['HAVE_UNISCRIBE']:
            env.Append(
                CPPDEFINES=['HAVE_UNISCRIBE'],
                LIBS=[
                    'usp10',
                    'gdi32',
                    'rpcrt4'
                ])
            project_sources += ['repo/src/hb-uniscribe.cc']
            project_headers += ['repo/src/hb-uniscribe.h']

        if env['HAVE_DIRECTWRITE']:
            env.Append(
                CPPDEFINES=['HAVE_DIRECTWRITE'],
                LIBS=[
                    'dwrite',
                    'rpcrt4'
                ])
            project_sources += ['repo/src/hb-directwrite.cc']
            project_headers += ['repo/src/hb-directwrite.h']
    headercom = None
    sourcecom = None

    ConfigureEnv(env)
    env.Append(CPPPATH='repo/src')

    if env['HAVE_GOBJECT']:

        glibmkenum_path = None
        perl_path = None
        glibmkenum_cmd = None

        bin_ext = ""
        if sys.platform == 'win32':
            bin_ext = ".exe"

        for path in os.environ["PATH"].split(os.pathsep):
            if os.access(os.path.join(path, 'glib-mkenums'), os.X_OK):
                glibmkenum_path = os.path.join(path, 'glib-mkenums')
            if os.access(os.path.join(path, 'perl' + bin_ext), os.X_OK):
                perl_path = os.path.join(path, 'perl' + bin_ext)

        if glibmkenum_path:
            process = subprocess.Popen(
                [sys.executable, glibmkenum_path, '--version'], stdout=subprocess.PIPE, stderr=subprocess.PIPE)
            stdout, stderr = process.communicate()
            if process.returncode == 0:
                glibmkenum_cmd = [sys.executable, glibmkenum_path]
            elif perl_path:
                process = subprocess.Popen(
                    [perl_path, glibmkenum_path, '--version'], stdout=subprocess.PIPE, stderr=subprocess.PIPE)
                stdout, stderr = process.communicate()
                if process.returncode == 0:
                    glibmkenum_cmd = [perl_path, glibmkenum_path]
        else:
            p.ErrorPrint(
                "Requested build with gobjects, but couldn't find glib-mkenums")
            return

        if not glibmkenum_cmd:
            p.ErrorPrint(
                "Requested build with gobjects, but couldn't find glib-mkenums")
            return

        gobject_sources += ['repo/src/hb-gobject-structs.cc']
        gobject_gen_sources += ['repo/src/hb-gobject-enums.cc']
        gobject_structs_headers += ['repo/src/hb-gobject-structs.h']
        gobject_headers += gobject_structs_headers + \
            ['repo/src/hb-gobject.h']
        gobject_gen_headers += ['repo/src/hb-gobject-enums.h']

        def replace_enums(env, target, source):
            header_target = os.path.basename(target[0].abspath)
            f1 = open('repo/src/' + os.path.basename(target[0].abspath), 'w')
            f2 = open('repo/src/' + os.path.basename(target[0].abspath), 'r')
            contents = f2.read().replace('_t_get_type', '_get_type').replace('_T (', ' (')
            f2.close()
            f1.write(contents)
            f1.close()

        cmd = ' '.join(glibmkenum_cmd + [
            '--template', 'repo/src/${TARGET.file}.tmpl', 
            '--identifier-prefix', 'hb_',
            '--symbol-prefix', 'hb_gobject']) + " $SOURCES > $TARGET"

        def glib_mkenums_emitter(target, source, env):
            source = [file for file in source if not file.abspath.endswith('.tmpl')]
            target_file = 'repo/src/' + os.path.basename(target[0].abspath)
            # hacky: https://github.com/SCons/scons/issues/2908
            Depends(env['BUILD_DIR'] + '/build_harfbuzz-gobject_shared/' + target_file, target_file)
            Depends(env['BUILD_DIR'] + '/build_harfbuzz-gobject_static/' + target_file, target_file)
            return (target_file, source)

        env['BUILDERS']['glib_mkenums_headers'] = Builder(
            action = [SCons.Action.CommandAction(cmd), SCons.Action.FunctionAction(replace_enums, {})],
            emitter = glib_mkenums_emitter,
            suffix = '.h', src_suffix = '.tmpl')
        env['BUILDERS']['glib_mkenums_sources'] = Builder(
            action = [SCons.Action.CommandAction(cmd), SCons.Action.FunctionAction(replace_enums, {})],
            emitter = glib_mkenums_emitter,
            suffix = '.cc', src_suffix = '.tmpl')

        def generate_header(s, target, source, env):
            if not env['GLIBMKENUMS_HEADERS_DONE']:
                p.InfoPrint(" Generating glib-mkenums headers...")
                env['GLIBMKENUMS_HEADERS_DONE'] = True
        def generate_source(s, target, source, env):
            if not env['GLIBMKENUMS_SOURCE_DONE']:
                p.InfoPrint(" Generating glib-mkenums sources...")
                env['GLIBMKENUMS_SOURCE_DONE'] = True

        env.glib_mkenums_headers(
            'repo/src/hb-gobject-enums.h', 
            ['repo/src/hb-gobject-enums.h.tmpl'] + gobject_structs_headers + project_headers, 
            PRINT_CMD_LINE_FUNC=generate_header, 
            GLIBMKENUMS_HEADERS_DONE=False )

        env.glib_mkenums_sources(
            'repo/src/hb-gobject-enums.cc',  
            ['repo/src/hb-gobject-enums.cc.tmpl'] + gobject_headers + project_headers, 
            PRINT_CMD_LINE_FUNC=generate_source,
            GLIBMKENUMS_SOURCE_DONE=False) 

    if env['BUILD_SHARED']:
        if env['CC'] == 'cl':
            env.Append(CPPDEFINES=['HB_DLL_EXPORT'])

    if (sys.platform != 'win32' or 
        (env.Detect('gcc') and sys.platform == 'win32')):
        if env['HAVE_BSYMBOLIC']:
            env.Append(
                LINKFLAGS=['-Bsymbolic-functions']
            )
        if env['CC'] == 'gcc' or env['CC'] == 'clang':
            env.Append(
                CCFLAGS=[
                    '-fno-rtti', 
                    '-fno-exceptions',
                    '-fno-threadsafe-statics'
                ],
                LIBS=['m']
            )
        if env['HAVE_CPP11']:
            env.Append(CCFLAGS=['-std=c++11'])

    if sys.platform == 'win32' and not env['BUILD_SHARED']:
        env['HAVE_INTROSPECTION'] = False

    if env['BUILD_UTILS'] and FindCairo(env, conf_dir=env['BUILD_DIR']):
       
        utils = [
            ('hb-view', GetSources('HB_VIEW_sources', 'repo/util/Makefile.sources')),
            ('hb-shape', GetSources('HB_SHAPE_sources', 'repo/util/Makefile.sources')),
            ('hb-subset', GetSources('HB_SUBSET_CLI_sources', 'repo/util/Makefile.sources')),
            ('hb-ot-shape-closure', GetSources('HB_OT_SHAPE_CLOSURE_sources', 'repo/util/Makefile.sources')),
        ]

        for util, sources in utils:

            source_files = []
            header_files = []
            SortSources(
                source_files, 
                header_files, 
                sources
            )
            util_env, util_bin = SetupBuildEnv(
                env,
                progress,
                'exec',
                util,
                source_files,
                env['BUILD_DIR'] + '/utils/' + util,
                'install/bin')
            util_env.Append(
                LIBPATH=[env['BUILD_DIR'] + '/build_harfbuzz_shared'],
                LIBS=['harfbuzz'],
                CPPDEFINES=[
                    ('PACKAGE_NAME', '\\"HarfBuzz\\"'),
                    ('PACKAGE_VERSION', env['HB_VERSION'].replace('.', '')),
                ])
            if util == 'hb-subset':
                util_env.Append(
                    LIBPATH=[env['BUILD_DIR'] + '/build_harfbuzz-subset_shared'],
                    LIBS=['harfbuzz-subset']
                )


    if env['HAVE_INTROSPECTION']:

        gircompiler_path = None
        girscanner_path = None
        for path in os.environ["PATH"].split(os.pathsep):
            if os.access(os.path.join(path, 'g-ir-scanner'), os.X_OK):
                girscanner_path = os.path.join(path, 'g-ir-scanner')
            if os.access(os.path.join(path, 'g-ir-compiler'), os.X_OK):
                gircompiler_path = os.path.join(path, 'g-ir-compiler')

        if not girscanner_path:
            p.ErrorPrint("Can't build introspection because no g-ir-scanner in path.")

        if not gircompiler_path:
            p.ErrorPrint("Can't build introspection because no g-ir-compiler in path.")

        introspection = (
            project_headers + 
            project_sources + 
            gobject_gen_sources + 
            gobject_gen_headers +
            gobject_sources +
            gobject_headers)

        def write_introspection_list(target, source, env):
            contents = ""
            for file in source:
                contents += file.path + '\n'
            f1 = open('repo/src/hb_gir_list', 'w')
            f1.write(contents)
            f1.close()

        def generate_introspection_list(s, target, source, env):
            p.InfoPrint(" Generating Introspection list...")

        env.Command(
            'repo/src/hb_gir_list', 
            introspection, 
            write_introspection_list,
            PRINT_CMD_LINE_FUNC=generate_introspection_list
            )

        scanner_cmd_str= [
            girscanner_path,
            '--warn-all', '--no-libtool', '--verbose',
            '-n', 'hb',
            '--namespace=HarfBuzz',
            '--nsversion=0.0',
            '--identifier-prefix=hb_',
            '--include', 'GObject-2.0',
            '--pkg-export=harfbuzz',
            '--cflags-begin'
        ]

        scanner_cmd_str += env['CCFLAGS']
        scanner_cmd_str += [ '-I' + include for include in env['CPPPATH']]
        scanner_cmd_str += ['-Irepo/src']

        for define in env['CPPDEFINES']:
            if type(define) is list or type(define) is tuple:
                scanner_cmd_str += [ '-D' + define[0] + '=' + define[1]]
            else:
                scanner_cmd_str += [ '-D' + define ]

        scanner_cmd_str += [
            '-DHB_H',
            '-DHB_H_IN',
            '-DHB_OT_H',
            '-DHB_OT_H_IN',
            '-DHB_AAT_H',
            '-DHB_AAT_H_IN',
            '-DHB_GOBJECT_H',
            '-DHB_GOBJECT_H_IN',
            '-DHB_EXTERN=',
            '--cflags-end',
            '--library=harfbuzz-gobject',
            '--library=harfbuzz'
        ]

        # --extra-library not supported option?
        #scanner_cmd_str += [ '--extra-library=' + flag for flag in env['LIBS'] if not flag.startswith('harfbuzz')]
        
        scanner_cmd_str += [
            '-L' + env['PROJECT_DIR'] + '/install/shared', 
            '--filelist', 
            'repo/src/hb_gir_list',
            '-o', 'install/shared/HarfBuzz-0.0.gir',
            '>', env['BUILD_DIR'] + '/build_harfbuzz-gobject_shared/girscanner.txt', '2>&1']

        def run_introspection_scanner(s, target, source, env):
            f1 = open(env['BUILD_DIR'] + '/build_harfbuzz-gobject_shared/girscanner_command.txt', 'w')
            f1.write(s)
            f1.close()
            p.InfoPrint(" Generating Introspection gir...")

        env.Command(
            'install/shared/HarfBuzz-0.0.gir', 
            'repo/src/hb_gir_list', 
            ' '.join(scanner_cmd_str),
            PRINT_CMD_LINE_FUNC=run_introspection_scanner
        )

        def run_introspection_compiler(s, target, source, env):
            f1 = open(env['BUILD_DIR'] + '/build_harfbuzz-gobject_shared/gircompiler_command.txt', 'w')
            f1.write(s)
            f1.close()
            p.InfoPrint(" Compiling Introspection typelib...")

        compiler_cmd_str = [
            gircompiler_path,
            '--verbose', '--debug',
            '--includedir', 'repo/src',
            'install/shared/HarfBuzz-0.0.gir',
            '-o', 'install/shared/HarfBuzz-0.0.typelib',
            '>', env['BUILD_DIR'] + '/build_harfbuzz-gobject_shared/gircompiler.txt', '2>&1'
        ]

        env.Command(
            'install/shared/HarfBuzz-0.0.typelib', 
            'install/shared/HarfBuzz-0.0.gir', 
            ' '.join(compiler_cmd_str),
            PRINT_CMD_LINE_FUNC=run_introspection_compiler
        )

    
    
    libraries = [{
        'name':'harfbuzz',
        'source': project_sources + project_extra_sources,
    }]

    if env['BUILD_SUBSET']:
        subset_project_sources = [
            source for source in subset_project_sources if source.endswith(".cc")]
        libraries.append({
            'name':'harfbuzz-subset',
            'source': subset_project_sources,
            'depends': 'harfbuzz'
        })

    if env['HAVE_GOBJECT']: 
        libraries.append({
            'name':'harfbuzz-gobject',
            'source':gobject_sources + gobject_gen_sources,
            'depends': 'harfbuzz'
        })
       
    for lib in libraries:

        static_env, static_lib = SetupBuildEnv(
            env,
            progress,
            'static',
            lib['name'],
            lib['source'],
            env['BUILD_DIR'] + '/build_' + lib['name'] + '_static',
            'install/static')

        if lib.get('depends'):
            static_env.Append(
                LIBS=[lib['depends']],
                LIBPATH=[env['PROJECT_DIR'] + '/' + static_env['BUILD_DIR'] + '/build_' + lib['depends'] + '_static'])
        
        if env['BUILD_SHARED']:
            shared_env, shared_lib = SetupBuildEnv(
                env,
                progress,
                'shared',
                lib['name'],
                lib['source'],
                env['BUILD_DIR'] + '/build_' + lib['name'] + '_shared',
                'install/shared')

            if env['CC'] != 'cl':
                shared_env.Append(
                    CCFLAGS=['-fvisibility-inlines-hidden'])

            if lib.get('depends'):
                shared_env.Append(
                    LIBS=[lib['depends']],
                    LIBPATH=[env['PROJECT_DIR'] + '/' + env['BUILD_DIR'] + '/build_' + lib['depends'] + '_shared'])

    if env['BUILD_TESTS']:

        tests = [
            'main',
            'test',
            'test-gsub-would-substitute', 
            'test-gpos-size-params',
            'test-buffer-serialize',
            'hb-ot-tag', 
            'test-unicode-ranges'
        ]

        for test in tests:
            test_env, test_bin = SetupBuildEnv(
                env,
                progress,
                'exec',
                test,
                ['repo/src/' + test + ".cc"],
                env['BUILD_DIR'] + '/tests',
                'install/bin')
            if test == 'hb-ot-tag':
                test_env.Append(CPPDEFINES=['MAIN'])
            test_env.Append(
                LIBS=['harfbuzz'],
                LIBPATH=[env['PROJECT_DIR'] + '/' + env['BUILD_DIR'] + '/build_harfbuzz_shared'])

        if (sys.platform != 'win32' or 
            (env.Detect('gcc') and sys.platform == 'win32')):

            if env['BUILD_SHARED']:

                def generate_def_file(s, target, source, env):
                    p.InfoPrint(" Generating harfbuzz.def...")

                def run_tests(s, target, source, env):
                    for test in [os.path.basename(arg)for arg in s.split(' ') if os.path.basename(arg).endswith('.sh')]:
                        p.InfoPrint(' Running ' + test + ' test...')

                env.Command(
                    env['BUILD_DIR'] + '/build_harfbuzz_shared/harfbuzz.def', 
                    project_headers, 
                    sys.executable + ' repo/src/gen-def.py $TARGET $SOURCES',
                    PRINT_CMD_LINE_FUNC=generate_def_file )
                
                
                env.Command(
                    env['BUILD_DIR'] + '/tests/check-static-inits.out', 
                    env['BUILD_DIR'] + '/build_harfbuzz_static/libharfbuzz.a',
                    'export libs=. && cd ' + env['BUILD_DIR'] + '/build_harfbuzz_static/repo/src ' 
                        + env['PROJECT_DIR'] + '/repo/src/check-static-inits.sh > ' 
                        + env['PROJECT_DIR'] + '/$TARGET 2>&1',
                    PRINT_CMD_LINE_FUNC=run_tests)

                #env.Command(
                #    env['BUILD_DIR'] + '/tests/check-libstdc++.out', 
                #    ['install/shared/libharfbuzz.so','install/shared/libharfbuzz-gobject.so'],
                #    'export libs=. && cd install/shared && ' 
                #        + env['PROJECT_DIR'] + '/repo/src/check-libstdc++.sh > ' 
                #        + env['PROJECT_DIR'] + '/$TARGET 2>&1',
                #    PRINT_CMD_LINE_FUNC=run_tests)
                
    Progress(progress, interval=1)

    atexit.register(display_build_status, env['PROJECT_DIR'], start_time)

def ConfigureEnv(env):

    p = ColorPrinter()

    if not env.GetOption('clean'):

        if env['RECONF']:
            os.remove(env['BUILD_DIR'] + '/build.conf')

        vars = Variables(env['BUILD_DIR'] + '/build.conf')
        vars.Update(env)
        vars.Save(env['BUILD_DIR'] + '/build.conf', env)

        configureString = ""

        if configureString == "":
            configureString = " Configuring... "
        else:
            configureString = " Configuring with " + configureString

        p.InfoPrint(configureString)

        # ruins logs so turning it off
        #SCons.Script.Main.progress_display.set_mode(1)
        
        conf = Configure(env, conf_dir=env['BUILD_DIR'] + "/conf_tests", log_file=env['BUILD_DIR'] + "/conf.log",
                        custom_tests={  
                            'CheckLargeFile64': CheckLargeFile64,
                            'CheckFseeko': CheckFseeko,
                            'CheckSizeT': CheckSizeT,
                            'CheckSizeTLongLong': CheckSizeTLongLong,
                            'CheckSizeTPointerSize': CheckSizeTPointerSize,
                            'CheckSharedLibrary': CheckSharedLibrary,
                            'CheckUnistdH': CheckUnistdH,
                            'CheckSolarisAtomics': CheckSolarisAtomics,
                            'CheckIntelAtomicPrimitives': CheckIntelAtomicPrimitives,
                            'CheckFunc' : CheckFunc,
                            'CheckHeader' : CheckHeader,
                            'CheckBSymbolic' : CheckBSymbolic,
                            'CheckStdCpp11' : CheckStdCpp11
                        })
     
        # with open('repo/zconf.h.in', 'r') as content_file:
        #    conf.env["ZCONFH"] = str(content_file.read())

        conf.CheckSharedLibrary()

        env['HAVE_BSYMBOLIC'] = conf.CheckBSymbolic()
        env['HAVE_CPP11'] = conf.CheckStdCpp11()

        checks_funcs = [
        
            'mprotect',
            'sysconf',
            'getpagesize',
            'mmap',
            'isatty',
            'newlocale',
            'strtod_l',
            'round'
        ]

        if env.get('HB_HAVE_FREETYPE'):
            checks_funcs += [
                'FT_Get_Var_Blend_Coordinates',
                'FT_Set_Var_Blend_Coordinates',
                'FT_Done_MM_Var'
            ]

        for func in checks_funcs:
            have_func = 'HAVE_' + func.upper()
            if(conf.CheckFunc(func)):
                env[have_func] = True
                env.Append(CPPDEFINES=[have_func])
            else:
                env[have_func] = False

        check_headers = [
            'unistd.h',
            'sys/mman.h',
            'xlocale.h',
            'stdbool.h'
        ]

        for header in check_headers:
            have_header = 'HAVE_' + header.upper().replace('/', '_').replace('.', '_')
            if(conf.CheckHeader(header, language='C')):
                env[have_header] = True
                env.Append(CPPDEFINES=[have_header])
            else:
                env[have_header] = False

        # conf.CheckExternalNames()
        if not conf.CheckSizeT():
            conf.CheckSizeTPointerSize(conf.CheckSizeTLongLong())
        if not conf.CheckLargeFile64():
            conf.CheckFseeko()

        conf.CheckUnistdH()
        conf.CheckSolarisAtomics()
        conf.CheckIntelAtomicPrimitives()

        # ruins logs so turning it off
        #SCons.Script.Main.progress_display.set_mode(0)

        env = conf.Finish()

    if("linux" in platform.system().lower()):

        debugFlag = ""
        if(env['DEBUG_BUILD']):
            debugFlag = "-g"

        env.Append(CPPPATH=[
        ])

        env.Append(CCFLAGS=[
            debugFlag,
            '-O3',
            # '-fPIC',
            # "-rdynamic",
        ])

        env.Append(CXXFLAGS=[
            # "-std=c++11",
        ])

        env.Append(LDFLAGS=[
            debugFlag,
            # "-rdynamic",
        ])

        env.Append(LIBPATH=[
            # "/usr/lib/gnome-settings-daemon-3.0/",
        ])

        env.Append(LIBS=[
            'm',
        ])

    elif("darwin" in platform.system().lower()):
        print("XCode project not implemented yet")
    elif("win" in platform.system().lower()):

        degugDefine = 'NDEBUG'
        debugFlag = "/O2"
        degug = '/DEBUG:NONE'
        debugRuntime = "/MD"
        libType = "Release"
        if(env['DEBUG_BUILD']):
            degugDefine = 'DEBUG'
            debugFlag = "/Od"
            degug = '/DEBUG:FULL'
            debugRuntime = "/MDd"
            libType = 'Debug'

        env.Append(
            CCFLAGS=[
                '/wd4244',
                '/wd4267'
            ],
            CPPDEFINES=[
                '_CRT_SECURE_NO_WARNINGS',
                '_CRT_NONSTDC_NO_WARNINGS',
            ],

        )

    env.Append(
        CPPDEFINES=['HAVE_FALLBACK']
    )

    return env


def ConfigPlatformIDE(env, sourceFiles, headerFiles, resources, program):
    if platform == "linux" or platform == "linux2":
        print("Eclipse C++ project not implemented yet")
    elif("darwin" in platform.system().lower()):
        print("XCode project not implemented yet")
    elif("win" in platform.system().lower()):
        variantSourceFiles = []
        for file in sourceFiles:
            variantSourceFiles.append(re.sub("^build", "../Source", file))
        variantHeaderFiles = []
        for file in headerFiles:
            variantHeaderFiles.append(re.sub("^Source", "../Source", file))
        buildSettings = {
            'LocalDebuggerCommand': os.path.abspath(env['PROJECT_DIR']).replace('\\', '/') + "/build/MyLifeApp.exe",
            'LocalDebuggerWorkingDirectory': os.path.abspath(env['PROJECT_DIR']).replace('\\', '/') + "/build/",

        }
        buildVariant = 'Release|x64'
        cmdargs = ''
        if(env['DEBUG_BUILD']):
            buildVariant = 'Debug|x64'
            cmdargs = '--debug_build'
        env.MSVSProject(target='VisualStudio/MyLifeApp' + env['MSVSPROJECTSUFFIX'],
                        srcs=variantSourceFiles,
                        localincs=variantHeaderFiles,
                        resources=resources,
                        buildtarget=program,
                        DebugSettings=buildSettings,
                        variant=buildVariant,
                        cmdargs=cmdargs)
    return env


def GetSources(var, file):
    with open(file) as f:
        sources = re.search(
            r'^' + var + r'\s=\s([^$]+)\\$', f.read(), re.MULTILINE).group(1).split('\\')
        sources = [line.strip() for line in sources if line != ""]
        sources = [os.path.dirname(file) + '/' + line for line in sources]
        return sources


def SortSources(source_list, header_list, mix_list):
    for source in mix_list:
        if source.endswith(".cc") or source.endswith(".c"):
            source_list.append(source)
        elif source.endswith(".hh") or source.endswith(".h"):
            header_list.append(source)
        else:
            p.ErrorPrint("Unhandled source: " + source)

def GetHarfbuzzVersion(env=None):
    with open('repo/configure.ac') as f:
        match = re.search(
            r'\[(([0-9]+)\.([0-9]+)\.([0-9]+))\]', f.read(), re.MULTILINE)
        if env:
            env['HB_VERSION'] = match.group(1).strip()
            env['HB_VERSION_MAJOR'] = match.group(2).strip()
            env['HB_VERSION_MINOR'] = match.group(3).strip()
            env['HB_VERSION_MICRO'] = match.group(4).strip()
        return match.group(1).strip()


def SetupInstalls(env):

    return env


def SetupOptions():

    AddOption(
        '--have-freetype',
        dest='option_have_freetype',
        action='store_true',
        default=False,
        help='Enable freetype interop helpers'
    )

    AddOption(
        '--have-graphite2',
        dest='option_have_graphite2',
        action='store_true',
        default=False,
        help='Enable Graphite2 complementary shaper'
    )

    AddOption(
        '--build-utils',
        dest='option_build_utils',
        action='store_true',
        metavar='BOOL',
        default=False,
        help='Build harfbuzz utils, needs cairo, freetype, and glib properly be installed'
    )

    AddOption(
        '--have-glib',
        dest='option_have_glib',
        action='store_true',
        metavar='BOOL',
        default=False,
        help='Enable glib unicode functions'
    )

    AddOption(
        '--have-icu',
        dest='option_have_icu',
        action='store_true',
        metavar='BOOL',
        default=False,
        help='Enable icu unicode functions'
    )

    AddOption(
        '--no-build-subset',
        dest='option_build_subset',
        action='store_false',
        metavar='BOOL',
        default=True,
        help="Don't build harfbuzz-subset"
    )

    AddOption(
        '--no-build-tests',
        dest='option_build_tests',
        action='store_false',
        metavar='BOOL',
        default=True,
        help="Don't Build harfbuzz tests"
    )

    AddOption(
        '--no-shared-libs',
        dest='option_build_shared',
        action='store_false',
        metavar='BOOL',
        default=True,
        help="Don't build shared libraries"
    )

    AddOption(
        '--have-gobject',
        dest='option_have_gobject',
        action='store_true',
        metavar='BOOL',
        default=False,
        help='Enable GObject Bindings'
    )

    AddOption(
        '--have-introspection',
        dest='option_have_introspection',
        action='store_true',
        metavar='BOOL',
        default=False,
        help='Enable building introspection (.gir/.typelib) files'
    )

    AddOption(
        '--check',
        dest='option_check',
        action='store_true',
        metavar='BOOL',
        default=False,
        help='Do a configuration suitable for testing (shared library and enable all options)'
    )

    AddOption(
        '--verbose',
        dest='option_verbose',
        action='store_true',
        metavar='BOOL',
        default=False,
        help='View compiler output.'
    )

    AddOption(
        '--debug-build',
        dest='option_debug',
        action='store_true',
        metavar='BOOL',
        default=False,
        help='Build Debug'
    )

    AddOption(
        '--reconfigure',
        dest='option_reconfigure',
        action='store_true',
        metavar='BOOL',
        default=False,
        help='Clear out configuration and reconfigure with new options.'
    )

    if sys.platform == 'win32':
        AddOption(
            '--have-uniscribe',
            dest='option_have_uniscribe',
            action='store_true',
            metavar='BOOL',
            default=False,
            help='Enable Uniscribe shaper backend on Windows'
        )

        AddOption(
            '--have-directwrite',
            dest='option_have_directwrite',
            action='store_true',
            metavar='BOOL',
            default=False,
            help='Enable DirectWrite shaper backend on Windows'
        )

    if sys.platform == 'darwin':
        AddOption(
            '--have-coretext',
            dest='option_have_coretext',
            action='store_true',
            metavar='BOOL',
            default=False,
            help='Enable CoreText shaper backend on macOS'
        )

   
    # ruins logs so turning it off
    #if(not GetOption('option_verbose')):
        #scons_ver = SCons.__version__
        #if int(scons_ver[0]) >= 3:
        #    SetOption('silent', 1)
        #SCons.Script.Main.progress_display.set_mode(0)


CreateNewEnv()
