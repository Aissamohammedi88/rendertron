// Copyright 2026 The Chromium Authors. All rights reserved.
// Use of this source code is governed by a BSD-style license.
// 
// NEXUS AST DEFENDER - High-Performance Bytecode Security Scanner
// Author: Aissa Mohammedi (DGK) - NEXUS-OPEN-3.0
// Mission: Stop malicious reflection and deserialization at the AST level.

package org.chromium.base.security;

import java.io.IOException;
import java.io.InputStream;
import java.util.ArrayList;
import java.util.HashSet;
import java.util.List;
import java.util.Set;

/**
 * The NexusAstDefender is designed to run as a background daemon during Chromium's 
 * build process (GN/Ninja) or at runtime within the Android WebView sandbox.
 * It parses compiled bytecode and flags dangerous patterns before they execute.
 */
public final class NexusAstDefender {

    // Whitelist of safe classes allowed to be instantiated via reflection.
    private static final Set<String> SAFE_REFLECTION_CLASSES = new HashSet<>();
    
    // Blacklist of known dangerous methods.
    private static final Set<String> FORBIDDEN_METHODS = new HashSet<>();

    static {
        // Populate safe reflection classes (e.g., Chrome's internal factories)
        SAFE_REFLECTION_CLASSES.add("org.chromium.base.Factory");
        SAFE_REFLECTION_CLASSES.add("org.chromium.content.browser.WebContents");

        // Populate forbidden methods that are common attack vectors.
        FORBIDDEN_METHODS.add("java/lang/Runtime.exec");
        FORBIDDEN_METHODS.add("java/lang/ProcessBuilder.start");
        FORBIDDEN_METHODS.add("java/io/ObjectInputStream.readObject");
        FORBIDDEN_METHODS.add("java/lang/Class.forName");
    }

    private final SecurityAuditListener auditListener;

    public NexusAstDefender(SecurityAuditListener listener) {
        this.auditListener = listener;
    }

    /**
     * Scans a compiled .class file stream for dangerous bytecode patterns.
     * @param classStream The input stream of the .class file.
     * @param className The name of the class being scanned.
     * @return true if the class is safe, false if it contains a threat.
     */
    public boolean scanClass(InputStream classStream, String className) throws IOException {
        if (classStream == null) {
            throw new IllegalArgumentException("Class stream cannot be null");
        }

        boolean isSafe = true;

        // Note: In a real implementation, we would use ASM (ObjectWeb) to parse the bytecode.
        // For this contribution, we simulate the bytecode analysis using string matching 
        // on the constant pool to keep the footprint minimal.
        
        // Read the entire class into memory (assume max 10MB for Chromium classes)
        byte[] buffer = new byte[10240];
        StringBuilder constantPool = new StringBuilder();
        int bytesRead;
        while ((bytesRead = classStream.read(buffer)) != -1) {
            for (int i = 0; i < bytesRead; i++) {
                constantPool.append((char) buffer[i]);
            }
        }

        // 1. Check for forbidden method invocations in the constant pool
        for (String forbidden : FORBIDDEN_METHODS) {
            if (constantPool.indexOf(forbidden) != -1) {
                isSafe = false;
                if (auditListener != null) {
                    auditListener.onThreatDetected(className, forbidden);
                }
                // In Chromium, we might want to throw a hard SecurityException here.
            }
        }

        // 2. Check for unsafe reflection usage
        if (constantPool.indexOf("java/lang/reflect/Method.invoke") != -1) {
            boolean hasSafeTarget = false;
            for (String safeClass : SAFE_REFLECTION_CLASSES) {
                if (constantPool.indexOf(safeClass) != -1) {
                    hasSafeTarget = true;
                    break;
                }
            }
            if (!hasSafeTarget) {
                isSafe = false;
                if (auditListener != null) {
                    auditListener.onThreatDetected(className, "Unsafe Reflection");
                }
            }
        }

        return isSafe;
    }

    /**
     * Interface for receiving security audit events.
     */
    public interface SecurityAuditListener {
        void onThreatDetected(String className, String threatSignature);
    }
}
