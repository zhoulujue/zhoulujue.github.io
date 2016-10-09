---
layout: post
title: Android 动态检查是否拥有某项权限的最佳实践
---

API Level 23 的 ContextCompact.checkSelfPermission() 在 小于 23 时，永远返回 PERMISSION_GRANTED

在 API Level = 23 时，ContextCompact.checkSelfPermission() 才返回正确的值

如果 targetAPI Level 在 23 以前，但是又要支持23，使用 PermissionChecker.checkSelfPermission() 是最靠谱的，

这个 PermissionChecker support-v4 lib 里新出的API

需要 在build.gradle 里

          compile ‘com.android.support:support-v4:23.1.1’

注意版本号，版本号较低的没有这个API

首先检查是否在安装时申请并授予了指定权限，如果没有再查询AppOps是否是在安装后被用户使用AppOps revoke 了权限。

如果没有记错的话，这个 AppOps 是android官方在4.3的时候引进的一个权限管理模块，可能后来觉得不好就给干掉了

然而现在在23上使用 PermissionChecker 检查的AppOps的结果和 实际的 “Runtime Permission”的权限授予结果一致，说明android在6.0上搞的所谓新权限机制”Runtime Permission” 本来就是复活了的 AppOps

    /**
     * Checks whether your app has a given permission and whether the app op
     * that corresponds to this permission is allowed.
     *
     * @param context Context for accessing resources.
     * @param permission The permission to check.
     * @return The permission check result which is either {@link #PERMISSION_GRANTED}
     *     or {@link #PERMISSION_DENIED} or {@link #PERMISSION_DENIED_APP_OP}.
     */
    public static int checkSelfPermission(@NonNull Context context,
            @NonNull String permission) {
        return checkPermission(context, permission, android.os.Process.myPid(),
                android.os.Process.myUid(), context.getPackageName());
    }

    /**
     * Checks whether a given package in a UID and PID has a given permission
     * and whether the app op that corresponds to this permission is allowed.
     *
     * @param context Context for accessing resources.
     * @param permission The permission to check.
     * @param pid The process id for which to check.
     * @param uid The uid for which to check.
     * @param packageName The package name for which to check. If null the
     *     the first package for the calling UID will be used.
     * @return The permission check result which is either {@link #PERMISSION_GRANTED}
     *     or {@link #PERMISSION_DENIED} or {@link #PERMISSION_DENIED_APP_OP}.
     */
    public static int checkPermission(@NonNull Context context, @NonNull String permission,
            int pid, int uid, String packageName) {
        if (context.checkPermission(permission, pid, uid) == PackageManager.PERMISSION_DENIED) {
            return PERMISSION_DENIED;
        }

        String op = AppOpsManagerCompat.permissionToOp(permission);
        if (op == null) {
            return PERMISSION_GRANTED;
        }

        if (packageName == null) {
            String[] packageNames = context.getPackageManager().getPackagesForUid(uid);
            if (packageNames == null || packageNames.length <= 0) {
                return PERMISSION_DENIED;
            }
            packageName = packageNames[0];
        }

        if (AppOpsManagerCompat.noteProxyOp(context, op, packageName)
                != AppOpsManagerCompat.MODE_ALLOWED) {
            return PERMISSION_DENIED_APP_OP;
        }

        return PERMISSION_GRANTED;
    }
