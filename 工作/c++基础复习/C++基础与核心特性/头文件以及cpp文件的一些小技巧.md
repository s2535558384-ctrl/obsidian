对于头文件和cpp文件在工作中使用的比较多了，这里写一些在实际项目中比较实用的技巧。
1.使用extern解决重定义技巧：如果在头文件中需要对一些声明的对象进行赋值 不妨尝试一下extern，在cpp中进行定义，这样可以防止链接时重定义的问题
2，#ifdef在工作中常用的技巧
// 网络编程中包含不同的头文件
#ifdef _WIN32
    #include <winsock2.h>
#elif defined(__linux__)
    #include <sys/socket.h>
    #include <netinet/in.h>
#endif
文件选择
QString getConfigPath() {
    QString path;
#ifdef Q_OS_WIN
    path = qgetenv("APPDATA"); // Windows 使用 AppData 目录
#elif defined(Q_OS_LINUX)
    path = QDir::homePath() + "/.config"; // Linux 使用 ~/.config
#elif defined(Q_OS_MACOS)
    path = QDir::homePath() + "/Library/Preferences"; // macOS 路径
#else
    path = QDir::currentPath();
#endif
    return path;
}

调试使用
void myDebugFunction(const QString &msg) {
#if defined(QT_DEBUG) // 或者自定义的宏，如 MY_DEBUG
    qDebug() << "[DEBUG]" << msg;
#else
    Q_UNUSED(msg) // 发布版本中避免参数未使用的警告
#endif
}