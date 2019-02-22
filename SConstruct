import os
import sys
import glob
import re
import time
import datetime
import atexit
import platform

import SCons.Action
import SCons.Script.Main

from BuildUtils.SconsUtils import SetBuildJobs, SetupBuildEnv, ProgressCounter
from BuildUtils.ColorPrinter import ColorPrinter
from BuildUtils.FindPackages import FindFreetype

def CreateNewEnv():

    SetupOptions()

    env = Environment(
        DEBUG_BUILD     = GetOption('option_debug'),
        VERBOSE_COMPILE = GetOption('option_verbose'),
    )

    env['PROJECT_DIR'] = os.path.abspath(Dir('.').abspath).replace('\\', '/')
    SetBuildJobs(env)

    GetHarfbuzzVersion(env)

    FindFreetype(env)

    exit()
    GetSources('HB_BASE_sources', 'repo/src/Makefile.sources')

    

    base_source_files = [
        'repo/adler32.c',
        'repo/compress.c',
        'repo/crc32.c',
        'repo/deflate.c',
        'repo/gzclose.c',
        'repo/gzlib.c',
        'repo/gzread.c',
        'repo/gzwrite.c',
        'repo/inflate.c',
        'repo/infback.c',
        'repo/inftrees.c',
        'repo/inffast.c',
        'repo/trees.c',
        'repo/uncompr.c',
        'repo/zutil.c',
    ]

    base_header_files = [

    ]

    distrib_header_files = [

    ]

    env = ConfigureEnv(env)
    env.Append(CPPPATH=['.'])
    prog_name = 'z'
    prog_static_name = prog_name
    if("Windows" in platform.system()):
        prog_static_name = prog_name + "_static"

    progress = ProgressCounter()

    static_env, z_static = SetupBuildEnv(env, progress, 'static', prog_static_name, base_source_files, 'build/build_static', 'deploy')
    shared_env, z_shared = SetupBuildEnv(env, progress, 'shared', prog_name, base_source_files, 'build/build_shared', 'deploy')

    if("Windows" in platform.system()):
        shared_env.Append(CPPDEFINES=['ZLIB_DLL'])

    Progress(progress, interval=1)

    zlib_static_lib = env.subst('$LIBPREFIX') + prog_static_name + env.subst('$LIBSUFFIX')

    if(not env['COVER']):
        example_env, example_bin = SetupBuildEnv(env, progress, 'exec', 'example', ['repo/test/example.c'], 'build/build_static', 'build')
        minizip_env, minizip_bin = SetupBuildEnv(env, progress, 'exec', 'minigzip', ['repo/test/minigzip.c'], 'build/build_static', 'build')

        example_env.Append(CPPPATH=[Dir('repo').abspath], LIBS=[File('./deploy/' + zlib_static_lib)])
        minizip_env.Append(CPPPATH=[Dir('repo').abspath], LIBS=[File('./deploy/' + zlib_static_lib)])

    else:
        infcover_env, infcover_bin = SetupBuildEnv(env, progress, 'exec', 'infcover', ['repo/test/infcover.c'], 'build/build_static', 'build')
        infcover_env.Append(CPPPATH=[Dir('repo').abspath], LIBS=[File('./deploy/' + zlib_static_lib)])

   
    #env = SetupInstalls(env)
    #env = ConfigPlatformIDE(env, source_files, headerFiles, resourceFiles, prog)

    
def ConfigureEnv(env):

    p = ColorPrinter()

    def CheckLargeFile64(context):
        context.Message(p.ConfigString('Checking for off64_t... ') )

        prev_defines = ""
        if('CPPDEFINES' in context.env):
            prev_defines = context.env['CPPDEFINES']

        context.env.Append(CPPDEFINES=['_LARGEFILE64_SOURCE=1'])
        result = context.TryCompile("""
            #include <sys/types.h>
            off64_t dummy = 0;
        """, 
        '.c')
        if not result:
            context.env.Replace(CPPDEFINES = prev_defines)
        context.Result(result)
        return result

    def CheckFseeko(context):
        context.Message(p.ConfigString('Checking for fseeko... ') )
        result = context.TryCompile("""
            #include <stdio.h>
            int main(void) {
                fseeko(NULL, 0, 0);
                return 0;
            }
        """, 
        '.c')
        if not result:
            context.env.Append(CPPDEFINES=['NO_FSEEKO'])
        context.Result(result)
        return result

    def CheckSizeT(context):
        context.Message(p.ConfigString('Checking for size_t... ') )
        result = context.TryCompile("""
            #include <stdio.h>
            #include <stdlib.h>
            size_t dummy = 0;
        """, 
        '.c')
        context.Result(result)
        return result
    
    def CheckSizeTLongLong(context):
        context.Message(p.ConfigString('Checking for long long... ') )
        result = context.TryCompile("""
            long long dummy = 0;
        """, 
        '.c')
        context.Result(result)
        return result

    def CheckSizeTPointerSize(context, longlong_result):
        result = []
        context.Message(p.ConfigString('Checking for a pointer-size integer type... ') )
        if longlong_result:
            result = context.TryRun("""
                #include <stdio.h>
                int main(void) {
                    if (sizeof(void *) <= sizeof(int)) puts("int");
                    else if (sizeof(void *) <= sizeof(long)) puts("long");
                    else puts("z_longlong");
                    return 0;
                }
            """,
            '.c')
        else:
            result = context.TryRun("""
                #include <stdio.h>
                int main(void) {
                    if (sizeof(void *) <= sizeof(int)) puts("int");
                    else puts("long");
                    return 0;
                }
            """, 
            '.c')
        
        if result[1] == "":
            context.Result("Failed.")
            return False
        else:
            context.env.Append(CPPDEFINES=['NO_SIZE_T='+result[1]])
            context.Result(result[1] + ".")
            return True

    def CheckSharedLibrary(context):
        context.Message(p.ConfigString( 'Checking for shared library support... ') )
        result = context.TryBuild(context.env.SharedLibrary, """
            extern int getchar();
            int hello() {return getchar();}
        """,
        '.c')

        context.Result(result)
        return result

    def CheckUnistdH(context):
        context.Message(p.ConfigString('Checking for unistd.h... ') )
        result = context.TryCompile("""
            #include <unistd.h>
            int main() { return 0; }
        """, 
        '.c')
        if result:
            context.env["ZCONFH"] = re.sub(
                r"def\sHAVE_UNISTD_H(.*)\smay\sbe", r" 1\1 was", context.env["ZCONFH"],re.MULTILINE)  
        context.Result(result)
        return result

   

    def CheckStrerror(context):
        context.Message(p.ConfigString('Checking for strerror... ') )
        result = context.TryCompile("""
            #include <string.h>
            #include <errno.h>
            int main() { return strlen(strerror(errno)); }
        """, 
        '.c')
        if not result:
            context.env.Append(CPPDEFINES=['NO_STRERROR'])
        context.Result(result)
        return result
        
    def CheckStdargH(context):
        context.Message(p.ConfigString('Checking for stdarg.h... ') )
        result = context.TryCompile("""
            #include <stdarg.h>
            int main() { return 0; }
        """, 
        '.c')
        if result:
            context.env["ZCONFH"] = re.sub(
                r"def\sHAVE_STDARG_H(.*)\smay\sbe", r" 1\1 was", context.env["ZCONFH"],re.MULTILINE)  
        context.Result(result)
        return result

    def AddZPrefix(context):
        context.Message(p.ConfigString('Using z_ prefix on all symbols... ') )
        result = context.env['ZPREFIX'] 
        if result:
            context.env["ZCONFH"] = re.sub(
                r"def\sZ_PREFIX(.*)\smay\sbe", r" 1\1 was", context.env["ZCONFH"],re.MULTILINE)  
        context.Result(result)
        return result

    def AddSolo(context):
        context.Message(p.ConfigString('Using Z_SOLO to build... ') )
        result = context.env['SOLO'] 
        if result:
            context.env["ZCONFH"] = re.sub(
                r"\#define\sZCONF_H", r"#define ZCONF_H\n#define Z_SOLO", context.env["ZCONFH"],re.MULTILINE)  
        context.Result(result)
        return result

    def AddCover(context):
        context.Message(p.ConfigString('Using code coverage flags... ') )
        result = context.env['COVER'] 
        if result:
            context.env.Append(CCFLAGS=[
                '-fprofile-arcs', 
                '-ftest-coverage',
            ])
            context.env.Append(LINKFLAGS=[
                '-fprofile-arcs', 
                '-ftest-coverage',
            ])
            context.env.Append(LIBS=[
                'gcov', 
            ])
        if not context.env['GCC_CLASSIC'] == "":
            context.env.Replace(CC = context.env['GCC_CLASSIC'])
        context.Result(result)
        return result
    
    def CheckVsnprintf(context):
        context.Message(p.ConfigString("Checking whether to use vs[n]printf() or s[n]printf()... ") )
        result = context.TryCompile("""
            #include <stdio.h>
            #include <stdarg.h>
            #include "zconf.h"
            int main()
            {
            #ifndef STDC
                choke me
            #endif
                return 0;
            }
        """, 
        '.c')
        if result:
            context.Result("using vs[n]printf().")
        else:
            context.Result("using s[n]printf().")
        return result

    def CheckVsnStdio(context):
        context.Message(p.ConfigString("Checking for vsnprintf() in stdio.h... ") )
        result = context.TryCompile("""
            #include <stdio.h>
            #include <stdarg.h>
            int mytest(const char *fmt, ...)
            {
                char buf[20];
                va_list ap;
                va_start(ap, fmt);
                vsnprintf(buf, sizeof(buf), fmt, ap);
                va_end(ap);
                return 0;
            }
            int main()
            {
                return (mytest("Hello%d\\n", 1));
            }
        """, 
        '.c')
        context.Result(result)
        if not result:
            context.env.Append(CPPDEFINES=["NO_vsnprintf"])
            print(p.ConfigString("  WARNING: vsnprintf() not found, falling back to vsprintf(). zlib") )
            print(p.ConfigString("  can build but will be open to possible buffer-overflow security") )
            print(p.ConfigString("  vulnerabilities.") )
        return result

    def CheckVsnprintfReturn(context):
        context.Message(p.ConfigString("Checking for return value of vsnprintf()... ") )
        result = context.TryCompile("""
            #include <stdio.h>
            #include <stdarg.h>
            int mytest(const char *fmt, ...)
            {
                int n;
                char buf[20];
                va_list ap;
                va_start(ap, fmt);
                n = vsnprintf(buf, sizeof(buf), fmt, ap);
                va_end(ap);
                return n;
            }
            int main()
            {
                return (mytest("Hello%d\\n", 1));
            }
        """, 
        '.c')
        context.Result(result)
        if not result:
            context.env.Append(CPPDEFINES=["HAS_vsnprintf_void"])
            print(p.ConfigString("  WARNING: apparently vsnprintf() does not return a value. zlib") )
            print(p.ConfigString("  can build but will be open to possible string-format security") )
            print(p.ConfigString("  vulnerabilities.") )
        return result

    def CheckVsprintfReturn(context):
        context.Message(p.ConfigString( "Checking for return value of vsnprintf()... ") )
        result = context.TryCompile("""
            #include <stdio.h>
            #include <stdarg.h>
            int mytest(const char *fmt, ...)
            {
                int n;
                char buf[20];
                va_list ap;
                va_start(ap, fmt);
                n = vsprintf(buf, fmt, ap);
                va_end(ap);
                return n;
            }
            int main()
            {
                return (mytest("Hello%d\\n", 1));
            }
        """, 
        '.c')
        context.Result(result)
        if not result:
            context.env.Append(CPPDEFINES=["HAS_vsprintf_void"])
            print(p.ConfigString("  WARNING: apparently vsprintf() does not return a value. zlib") )
            print(p.ConfigString("  can build but will be open to possible string-format security") )
            print(p.ConfigString("  vulnerabilities.") )
        return result

    def CheckSnStdio(context):
        context.Message(p.ConfigString("Checking for snprintf() in stdio.h... ") )
        result = context.TryCompile("""
            #include <stdio.h>
            int mytest()
            {
                char buf[20];
                snprintf(buf, sizeof(buf), "%s", "foo");
                return 0;
            }
            int main()
            {
                return (mytest());
            }
        """, 
        '.c')
        context.Result(result)
        if not result:
            context.env.Append(CPPDEFINES=["NO_snprintf"])
            print(p.ConfigString("  WARNING: snprintf() not found, falling back to sprintf(). zlib") )
            print(p.ConfigString("  can build but will be open to possible buffer-overflow security") )
            print(p.ConfigString("  vulnerabilities.") )
        return result

    def CheckSnprintfReturn(context):
        context.Message(p.ConfigString("Checking for return value of snprintf()... ") )
        result = context.TryCompile("""
            #include <stdio.h>
            int mytest()
            {
                char buf[20];
                return snprintf(buf, sizeof(buf), "%s", "foo");
            }
            int main()
            {
                return (mytest());
            }
        """, 
        '.c')
        context.Result(result)
        if not result:
            context.env.Append(CPPDEFINES=["HAS_snprintf_void"])
            print(p.ConfigString("  WARNING: apparently snprintf() does not return a value. zlib") )
            print(p.ConfigString("  can build but will be open to possible string-format security") )
            print(p.ConfigString("  vulnerabilities.") )
        return result

    def CheckSprintfReturn(context):
        context.Message(p.ConfigString("Checking for return value of sprintf()... ") )
        result = context.TryCompile("""
            #include <stdio.h>
            int mytest()
            {
                char buf[20];
                return sprintf(buf, "%s", "foo");
            }
            int main()
            {
                return (mytest());
            }
        """, 
        '.c')
        context.Result(result)
        if not result:
            context.env.Append(CPPDEFINES=["HAS_sprintf_void"])
            print(p.ConfigString("  WARNING: apparently sprintf() does not return a value. zlib") )
            print(p.ConfigString("  can build but will be open to possible string-format security") )
            print(p.ConfigString("  vulnerabilities.") )
        return result

    def CheckHidden(context):
        context.Message(p.ConfigString("Checking for attribute(visibility) support... ") )
        result = context.TryCompile("""
            #define ZLIB_INTERNAL __attribute__((visibility ("hidden")))
            int ZLIB_INTERNAL foo;
            int main()
            {
                return 0;
            }
        """, 
        '.c')
        context.Result(result)
        if result:
            context.env.Append(CPPDEFINES=["HAVE_HIDDEN"])
        return result

    if not env.GetOption('clean'):
    
        if(GetOption('option_reconfigure')):
            os.remove('build.conf')

        vars = Variables('build.conf')
        
        #vars.Add('ZPREFIX', '' )
        #vars.Add('SOLO', '')
        #vars.Add('COVER', '')
        vars.Update(env)

        #if('--zprefix' in sys.argv): env['ZPREFIX'] = GetOption('option_zprefix')
        #if('--solo' in sys.argv):    env['SOLO']    = GetOption('option_solo')
        #if('--cover' in sys.argv):   env['COVER']   = GetOption('option_cover')

        vars.Save('build.conf', env)

        configureString = ""
        #if env['ZPREFIX']: configureString += "--zprefix "
        #if env['SOLO']:    configureString += "--solo "
        #if env['COVER']:   configureString += "--cover "

        if configureString == "": configureString = "Configuring... "
        else:                     configureString = "Configuring with " + configureString

        p.InfoPrint(configureString)

        SCons.Script.Main.progress_display.set_mode(1)

        conf = Configure(env, conf_dir = "build/conf_tests", log_file = "conf.log", 
            custom_tests = {
                'CheckLargeFile64'     : CheckLargeFile64, 
                'CheckFseeko'          : CheckFseeko,
                'CheckSizeT'           : CheckSizeT,
                'CheckSizeTLongLong'   : CheckSizeTLongLong,
                'CheckSizeTPointerSize': CheckSizeTPointerSize,
                'CheckSharedLibrary'   : CheckSharedLibrary,
                'CheckUnistdH'         : CheckUnistdH,
                'CheckStrerror'        : CheckStrerror,
                'CheckStdargH'         : CheckStdargH,
                'CheckVsnprintf'       : CheckVsnprintf,
                'CheckVsnStdio'        : CheckVsnStdio,
                'CheckVsnprintfReturn' : CheckVsnprintfReturn,
                'CheckVsprintfReturn'  : CheckVsprintfReturn,
                'CheckSnStdio'         : CheckSnStdio,
                'CheckSnprintfReturn'  : CheckSnprintfReturn,
                'CheckSprintfReturn'   : CheckSprintfReturn,
                'CheckHidden'          : CheckHidden,
                'AddZPrefix'           : AddZPrefix,
                'AddSolo'              : AddSolo,
                'AddCover'             : AddCover,
                })

        #with open('repo/zconf.h.in', 'r') as content_file:
        #    conf.env["ZCONFH"] = str(content_file.read())  

        conf.CheckSharedLibrary()

        checks_funcs = [
            'atexit',
            'mprotect',
            'sysconf',
            'getpagesize',
            'mmap',
            'isatty',
            'newlocale',
            'strtod_l',
            'round'
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
            if(conf.CheckCHeader(header)):
                env[have_header] = True
                env.Append(CPPDEFINES=[have_header])
            else:
                env[have_header] = False

        #conf.CheckExternalNames()
        #if not conf.CheckSizeT():
        #    conf.CheckSizeTPointerSize(conf.CheckSizeTLongLong())
        #if not conf.CheckLargeFile64():
        #    conf.CheckFseeko()
        
        #conf.CheckStrerror()
        #conf.CheckUnistdH()
        #conf.CheckStdargH()

        #if conf.env['ZPREFIX']: conf.AddZPrefix()
        #if conf.env['SOLO']:    conf.AddSolo()
        #if conf.env['COVER']:   conf.AddCover()

        #with open('repo/zconf.h', 'w') as content_file:
        #    content_file.write(conf.env["ZCONFH"])

        #if conf.CheckVsnprintf():
        #    if conf.CheckVsnStdio():
        #        conf.CheckVsnprintfReturn()
        #    else:
        #        conf.CheckVsprintfReturn()
        #else:
        #    if conf.CheckSnStdio():
        #        conf.CheckSnprintfReturn()
        #    else:
        #        conf.CheckSprintfReturn()

        #conf.CheckHidden()

        SCons.Script.Main.progress_display.set_mode(0)

        env = conf.Finish()
    
    if("linux" in platform.system().lower() ):

        debugFlag = ""
        if(env['DEBUG_BUILD']):
            debugFlag = "-g"

        env.Append(CPPPATH=[
        ])

        env.Append(CCFLAGS=[
            debugFlag,
            '-O3',
            #'-fPIC',
            #"-rdynamic",
        ])

        env.Append(CXXFLAGS= [
            #"-std=c++11",
        ])

        env.Append(LDFLAGS= [
            debugFlag,
            #"-rdynamic",
        ])

        env.Append(LIBPATH=[
            #"/usr/lib/gnome-settings-daemon-3.0/",
        ])

        env.Append(LIBS=[
            'm',
        ])

    elif("darwin" in platform.system().lower()):
        print("XCode project not implemented yet")
    elif("win" in platform.system().lower() ):

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
    elif("win" in platform.system().lower() ):
        variantSourceFiles = []
        for file in sourceFiles:
            variantSourceFiles.append(re.sub("^build", "../Source", file))
        variantHeaderFiles = []
        for file in headerFiles:
                variantHeaderFiles.append(re.sub("^Source", "../Source", file))
        buildSettings = {
            'LocalDebuggerCommand':os.path.abspath(env['PROJECT_DIR']).replace('\\', '/') + "/build/MyLifeApp.exe",
            'LocalDebuggerWorkingDirectory':os.path.abspath(env['PROJECT_DIR']).replace('\\', '/') + "/build/",
            
        }
        buildVariant = 'Release|x64'
        cmdargs = ''
        if(env['DEBUG_BUILD']):
            buildVariant = 'Debug|x64'
            cmdargs = '--debug_build'
        env.MSVSProject(target = 'VisualStudio/MyLifeApp' + env['MSVSPROJECTSUFFIX'],
                    srcs = variantSourceFiles,
                    localincs = variantHeaderFiles,
                    resources = resources,
                    buildtarget = program,
                    DebugSettings = buildSettings,
                    variant = buildVariant,
                    cmdargs = cmdargs)
    return env

def getSources(var, file):
    with open (file) as f:
        sources = re.search(r'^' + var + r'\s=\s([^$]+)\\$', f.read(), re.MULTILINE).group(1).split('\\')
        sources = [line.strip() for line in sources if line != ""]
        sources = [os.path.dirname(file) + '/' + line  for line in sources]
        return sources

def GetHarfbuzzVersion(env=None):
    with open('repo/configure.ac') as f:
        match = re.search(r'\[(([0-9]+)\.([0-9]+)\.([0-9]+))\]', f.read(), re.MULTILINE)
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
        '--no-builtin-ucdn',
        dest='option_builtin_ucdn',
        action='store_false',
        metavar='BOOL',
        default=True,
        help="Don't use HarfBuzz provided UCDN"
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
    
    if GetOption('option_build_utils'):
        SetOption('option_have_glib', True)
        SetOption('option_have_freetype', True)

    if GetOption('option_have_gobject'):
        SetOption('option_have_glib', True)

    if GetOption('option_have_introspection'):
        SetOption('option_have_gobject', True)
        SetOption('option_have_glib', True)

    if GetOption('option_check'):
        SetOption('option_build_shared', True)
        SetOption('option_build_utils', True)
        SetOption('option_builtin_ucdn', True)
        SetOption('option_have_icu', True)
        SetOption('option_have_glib', True)
        SetOption('option_have_gobject', True)
        SetOption('option_have_introspection', True)
        SetOption('option_have_freetype', True)
        SetOption('option_have_graphite2', True)
        if sys.platform == 'win32':
            SetOption('option_have_uniscribe', True)
            SetOption('option_have_directwrite', True)
        if sys.platform == 'darwin':
            SetOption('option_have_coretext', True)

    if(not GetOption('option_verbose')):
        scons_ver = SCons.__version__
        if int(scons_ver[0]) >= 3:
            SetOption('silent', 1)
        SCons.Script.Main.progress_display.set_mode(0)


CreateNewEnv()
