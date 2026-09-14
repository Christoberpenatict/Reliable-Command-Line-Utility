# Reliable-Command-Line-Utility

#include <iostream>
#include <string>
#include <vector>
#include <filesystem>
#include <csignal>
#include <cstdlib>

#if defined(_WIN32)
    #include <io.h>
    #define IS_TTY() _isatty(_fileno(stdout))
#else
    #include <unistd.h>
    #define IS_TTY() isatty(fileno(stdout))
#endif

namespace fs = std::filesystem;

enum ExitCode {
    SUCCESS = 0,
    ERROR_GENERAL = 1,
    ERROR_USAGE = 2,
    ERROR_INTERRUPTED = 130
};

struct Config {
    std::string target_path = "";
    bool json_output = false;
    bool verbose = false;
    bool show_help = false;
    bool show_version = false;
};

volatile std::sig_atomic_t g_signal_received = 0;

void signal_handler(int signal) {
    g_signal_received = signal;
}

void print_usage(const std::string& program_name) {
    std::cout << "Usage: " << program_name << " [OPTIONS] <TARGET_PATH>\n\n"
              << "A reliable, single-file C++ command-line utility.\n\n"
              << "Arguments:\n"
              << "  <TARGET_PATH>        Path to the target file or directory\n\n"
              << "Options:\n"
              << "  -j, --json           Output results in JSON format\n"
              << "  -v, --verbose        Enable detailed logging (sent to stderr)\n"
              << "  -h, --help           Show this help message\n"
              << "      --version        Show version information\n";
}

void print_version() {
    std::cout << "ReliableCLI version 1.0.0\n";
}

void log_verbose(const Config& config, const std::string& message) {
    if (config.verbose) {
        std::cerr << "[LOG] " << message << "\n";
    }
}

void log_error(const std::string& message) {
    std::cerr << "Error: " << message << "\n";
}

bool parse_arguments(int argc, char* argv[], Config& config) {
    std::vector<std::string> args(argv + 1, argv + argc);

  for (size_t i = 0; i < args.size(); ++i) {
        const std::string& arg = args[i];

   if (arg == "-h" || arg == "--help") {
            config.show_help = true;
            return true;
        } else if (arg == "--version") {
            config.show_version = true;
            return true;
        } else if (arg == "-j" || arg == "--json") {
            config.json_output = true;
        } else if (arg == "-v" || arg == "--verbose") {
            config.verbose = true;
        } else if (arg[0] == '-') {
            log_error("Unknown option: " + arg);
            return false;
      } else {
            if (config.target_path.empty()) {
                config.target_path = arg;
            } else {
                log_error("Unexpected additional argument: " + arg);
                return false;
            }
        }
    }

   if (config.target_path.empty() && !config.show_help && !config.show_version) {
        log_error("Missing required argument <TARGET_PATH>");
        return false;
    }
    return true;
}

int main(int argc, char* argv[]) {
    std::signal(SIGINT, signal_handler);
    std::signal(SIGTERM, signal_handler);

   Config config;

  if (!parse_arguments(argc, argv, config)) {
        std::cerr << "\nRun '" << argv[0] << " --help' for usage details.\n";
        return ExitCode::ERROR_USAGE;
    }

   if (config.show_help) {
        print_usage(argv[0]);
        return ExitCode::SUCCESS;
    }

   if (config.show_version) {
        print_version();
        return ExitCode::SUCCESS;
    }

  log_verbose(config, "Validating input path: " + config.target_path);

  fs::path path(config.target_path);
    if (!fs::exists(path)) {
        log_error("Path does not exist: " + config.target_path);
        return ExitCode::ERROR_GENERAL;
    }

  log_verbose(config, "Processing target path...");

  uintmax_t file_size = 0;
    try {
        if (fs::is_regular_file(path)) {
            file_size = fs::file_size(path);
        }
    } catch (const fs::filesystem_error& e) {
        log_error(e.what());
        return ExitCode::ERROR_GENERAL;
    }

  if (g_signal_received != 0) {
        std::cerr << "\nOperation aborted by user.\n";
        return ExitCode::ERROR_INTERRUPTED;
    }
    bool is_interactive_tty = IS_TTY();

   if (config.json_output) {
       std::cout << "{\n"
                  << "  \"path\": \"" << fs::absolute(path).string() << "\",\n"
                  << "  \"is_directory\": " << (fs::is_directory(path) ? "true" : "false") << ",\n"
                  << "  \"size_bytes\": " << file_size << "\n"
                  << "}\n";
    } else {
        if (is_interactive_tty) {
            std::cout << "\033[1;32mTarget Path:\033[0m " << fs::absolute(path).string() << "\n";
            std::cout << "\033[1;32mType:\033[0m        " << (fs::is_directory(path) ? "Directory" : "File") << "\n";
            std::cout << "\033[1;32mSize:\033[0m        " << file_size << " bytes\n";
        } else {
            std::cout << "Target Path: " << fs::absolute(path).string() << "\n";
            std::cout << "Type:        " << (fs::is_directory(path) ? "Directory" : "File") << "\n";
            std::cout << "Size:        " << file_size << " bytes\n";
        }
    }

   return ExitCode::SUCCESS;
}
