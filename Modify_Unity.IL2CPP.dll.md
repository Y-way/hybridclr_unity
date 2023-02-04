
# 2020

## HybridCLRConfig 同2021

## MethodSignatureWriter::FormatParameters2 同2021

## CodeWriterExtensions::WriteMethodWithMetadataInitialization

using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using Mono.Cecil;
using Unity.IL2CPP.Contexts;
using Unity.IL2CPP.Metadata;
using Unity.IL2CPP.Metadata.RuntimeTypes;
using Unity.IL2CPP.Naming;

namespace Unity.IL2CPP.CodeWriters
{
	// Token: 0x02000228 RID: 552
	public static partial class CodeWriterExtensions
	{
		// Token: 0x06001131 RID: 4401 RVA: 0x0005B774 File Offset: 0x00059974
		public static void WriteMethodWithMetadataInitialization(this IGeneratedMethodCodeWriter writer, string methodSignature, string methodFullName, Action<IGeneratedMethodCodeWriter, IRuntimeMetadataAccess> writeMethodBody, string uniqueIdentifier, MethodReference methodRef)
		{
			string identifier = uniqueIdentifier + "_MetadataUsageId";
			MethodMetadataUsage methodMetadataUsage = new MethodMetadataUsage();
			MethodUsage methodUsage = new MethodUsage();
			 if (methodRef != null && !Unity.IL2CPP.GenericSharing.GenericSharingAnalysis.CanShareMethod(writer.Context, methodRef) && HybridCLRConfig.Ins.IsDifferentialHybridAssembly(methodRef.Module.Name))
            {
                methodMetadataUsage.AddInflatedMethod(methodRef, false);
            }

## MethodBodyWriter::Generate

using System;
using System.Collections.Generic;
using System.Collections.ObjectModel;
using System.Diagnostics;
using Mono.Cecil;
using Mono.Cecil.Cil;
using Unity.Cecil.Awesome;
using Unity.Cecil.Awesome.CFG;
using Unity.IL2CPP.CodeWriters;
using Unity.IL2CPP.Contexts;
using Unity.IL2CPP.Contexts.Collectors;
using Unity.IL2CPP.Contexts.Services;
using Unity.IL2CPP.Debugger;
using Unity.IL2CPP.GenericSharing;
using Unity.IL2CPP.Naming;
using Unity.IL2CPP.StackAnalysis;

namespace Unity.IL2CPP
{
	// Token: 0x02000095 RID: 149
	[DebuggerDisplay("{_methodDefinition}")]
	public partial class MethodBodyWriter
	{
		// Token: 0x06000345 RID: 837
		public void Generate()
		{
			if (!this._methodDefinition.HasBody)
			{
				return;
			}
			if (HybridCLRConfig.Ins.IsDifferentialHybridAssembly(this._methodReference.Module.Name))
			{
				INamingService naming = this._context.Global.Services.Naming;
				if (!GenericSharingAnalysis.CanShareMethod(this._context, this._methodReference))
				{
					string str = naming.ForRuntimeMethodInfo(this._methodReference);
					this._writer.WriteStatement("method = " + str);
				}
				this._writer.WriteLine("if (method && method->isInterpterImpl)");
				this._writer.BeginBlock();
				this._writer.WriteStatement(string.Concat(new string[]
				{
					"typedef ",
					MethodSignatureWriter.FormatReturnType(this._context, this._context.ResolvedReturnType),
					" (*RedirectFunc)(",
					MethodSignatureWriter.FormatParameters2(this._context, this._methodReference, this._methodDefinition.HasThis ? ParameterFormat.WithTypeThisObject : ParameterFormat.WithType, true, true),
					")"
				}));
				string text = "((RedirectFunc)method->methodPointerCallByInterp)(" + MethodSignatureWriter.FormatParameters(this._context, this._methodReference, this._methodDefinition.HasThis ? ParameterFormat.WithName : ParameterFormat.WithNameNoThis, true) + ")";
				if (this._methodReference.ReturnType.IsNotVoid())
				{
					this._writer.WriteReturnStatement(text);
				}
				else
				{
					this._writer.WriteStatement(text);
					this._writer.WriteReturnStatement(null);
				}
				this._writer.EndBlock(false);
			}
			if (GenericsUtilities.CheckForMaximumRecursion(this._context, this._methodReference.DeclaringType as GenericInstanceType) || GenericsUtilities.CheckForMaximumRecursion(this._context, this._methodReference as GenericInstanceMethod))
			{
				this._writer.WriteStatement(Emit.RaiseManagedException("il2cpp_codegen_get_maximum_nested_generics_exception()", null));
				return;
			}
			this.WriteLocalVariables();
			if (this._context.Global.Parameters.EnableDebugger)
			{
				this.WriteDebuggerSupport();
				SequencePointInfo sequencePointInfo;
				if (this._sequencePointProvider.TryGetSequencePointAt(this._methodDefinition, -1, SequencePointKind.Normal, out sequencePointInfo))
				{
					this.WriteCheckSequencePoint(sequencePointInfo);
				}
				if (this._sequencePointProvider.TryGetSequencePointAt(this._methodDefinition, 16777215, SequencePointKind.Normal, out sequencePointInfo))
				{
					this.WriteCheckMethodExitSequencePoint(sequencePointInfo);
				}
				this.WriteCheckPausePoint(-1);
			}
			this._exceptionSupport = new ExceptionSupport(this._context, this._methodDefinition, this._cfg.Blocks, this._writer);
			this._exceptionSupport.Prepare();
			this.CollectUsedLabels();
			foreach (ExceptionHandler exceptionHandler in this._methodDefinition.Body.ExceptionHandlers)
			{
				if (exceptionHandler.CatchType != null)
				{
					this._writer.AddIncludeForTypeDefinition(this._typeResolver.Resolve(exceptionHandler.CatchType));
				}
			}
			foreach (GlobalVariable globalVariable in this._stackAnalysis.Globals)
			{
				this._writer.WriteVariable(this._typeResolver.Resolve(globalVariable.Type, true), globalVariable.VariableName);
			}
			foreach (ExceptionSupport.Node node in this._exceptionSupport.FlowTree.Children)
			{
				this.GenerateCodeRecursive(node);
			}
		}
	}
}




# 2021

=======================================

using System;
using System.Collections.Generic;
using Mono.Cecil;
using Unity.Cecil.Awesome;
using Unity.IL2CPP.CodeWriters;
using Unity.IL2CPP.Contexts;
using Unity.IL2CPP.Marshaling;
using System.Linq;

namespace Unity.IL2CPP
{
	// Token: 0x0200004B RID: 75
	public partial class MethodSignatureWriter
	{
		public static string FormatParameters2(ReadOnlyContext context, MethodReference method, ParameterFormat format, bool includeHiddenMethodInfo, bool useVoidPointerForThis)
		{   
			List<string> list = MethodSignatureWriter.ParametersForInternal(context, method, format, includeHiddenMethodInfo, useVoidPointerForThis, false).ToList<string>();
			if (list.Count != 0)
			{
				return list.AggregateWithComma();
			}
			return string.Empty;
		}
	}
}



===================================

using System;
using System.Collections.Generic;
using System.Collections.ObjectModel;
using System.Diagnostics;
using System.Linq;
using Unity.IL2CPP.CodeWriters;
using Unity.IL2CPP.Common;
using Unity.IL2CPP.Contexts;
using Unity.IL2CPP.Contexts.Services;
using Unity.IL2CPP.DataModel;
using Unity.IL2CPP.DataModel.Awesome.CFG;
using Unity.IL2CPP.Debugger;
using Unity.IL2CPP.MethodWriting;
using Unity.IL2CPP.Naming;
using Unity.IL2CPP.StackAnalysis;

namespace Unity.IL2CPP
{
    // Token: 0x02000076 RID: 118
    [DebuggerDisplay("{_methodDefinition}")]
    public partial class MethodBodyWriter
    {
        // Token: 0x060002EA RID: 746 RVA: 0x0001F724 File Offset: 0x0001D924
        public void Generate()
        {
            if (!this._methodDefinition.HasBody)
            {
                return;
            }
            if (HybridCLRConfig.Ins.IsDifferentialHybridAssembly(_methodReference.Module.Name))
            {
                INamingService naming = this._context.Global.Services.Naming;
                if (!this._methodReference.CanShare(this._context))
                {
                    string str = naming.ForRuntimeMethodInfo(this._context, this._methodReference);
                    this._writer.WriteStatement("method = " + str);
                }
                this._writer.WriteLine("if (method && method->isInterpterImpl)");
                this._writer.BeginBlock();
                this._writer.WriteStatement(string.Concat(new string[]
                {
                    "typedef ",
                    MethodSignatureWriter.FormatReturnType(this._context, this._context.ResolvedReturnType),
                    " (*RedirectFunc)(",
                    MethodSignatureWriter.FormatParameters2(this._context, this._methodReference, this._methodDefinition.HasThis ? ParameterFormat.WithTypeThisObject : ParameterFormat.WithType, true, true),
                    ")"
                }));
                string text = "((RedirectFunc)method->methodPointerCallByInterp)(" + MethodSignatureWriter.FormatParameters(this._context, this._methodReference, this._methodDefinition.HasThis ? ParameterFormat.WithName : ParameterFormat.WithNameNoThis, true) + ")";
                if (this._methodReference.ReturnType.IsNotVoid())
                {
                    this._writer.WriteReturnStatement(text);
                }
                else
                {
                    this._writer.WriteStatement(text);
                    this._writer.WriteReturnStatement(null);
                }
                this._writer.EndBlock(false);
            }
            if (GenericsUtilities.CheckForMaximumRecursion(this._context, this._methodReference) && this._methodReference != this._methodDefinition.FullySharedMethod)
            {
                if (this._context.Global.Parameters.FullGenericSharingMethodsAreBeingGenerated)
                {
                    List<string> list = new List<string>(this._methodDefinition.Parameters.Count + 2);
                    if (!this._methodDefinition.IsStatic)
                    {
                        list.Add("__this");
                    }
                    foreach (ParameterDefinition parameterDefinition in this._methodReference.Parameters)
                    {
                        list.Add(parameterDefinition.CppName);
                    }
                    this.WriteCallExpressionFor(this._methodReference, this._methodDefinition, MethodCallType.Normal, list, this._runtimeMetadataAccess.MethodMetadataFor(this._methodDefinition).OverrideHiddenMethodInfo("method"), false);
                    if (this._valueStack.Count == 1)
                    {
                        this._writer.WriteReturnStatement(this._valueStack.Pop().Expression);
                        return;
                    }
                }
                else
                {
                    this._writer.WriteStatement(Emit.RaiseManagedException("il2cpp_codegen_get_maximum_nested_generics_exception()", null));
                }
                return;
            }
            this.WriteLocalVariables();
            if (this._context.Global.Parameters.EnableDebugger)
            {
                this.WriteDebuggerSupport();
                SequencePointInfo sequencePointInfo;
                if (this._sequencePointProvider.TryGetSequencePointAt(this._methodDefinition, -1, SequencePointKind.Normal, out sequencePointInfo))
                {
                    this.WriteCheckSequencePoint(sequencePointInfo);
                }
                if (this._sequencePointProvider.TryGetSequencePointAt(this._methodDefinition, 16777215, SequencePointKind.Normal, out sequencePointInfo))
                {
                    this.WriteCheckMethodExitSequencePoint(sequencePointInfo);
                }
                this.WriteCheckPausePoint(-1);
            }
            this._exceptionSupport = new ExceptionSupport(this._context, this._methodDefinition, this._cfg.FlowTree, this._writer);
            this._exceptionSupport.Prepare();
            foreach (ExceptionHandler exceptionHandler in this._methodDefinition.Body.ExceptionHandlers)
            {
                if (exceptionHandler.CatchType != null)
                {
                    this._writer.AddIncludeForTypeDefinition(this._context, this._typeResolver.Resolve(exceptionHandler.CatchType));
                }
            }
            ReadOnlyDictionary<InstructionBlock, ResolvedInstructionBlock> instructionBlocks = this._resolvedMethodContext.Blocks.ToDictionary((ResolvedInstructionBlock b) => b.Block, (ResolvedInstructionBlock b) => b).AsReadOnly<InstructionBlock, ResolvedInstructionBlock>();
            foreach (GlobalVariable globalVariable in this._stackAnalysis.Globals)
            {
                this.WriteVariable(globalVariable.Type, globalVariable.VariableName, true);
            }
            foreach (Node node in this._exceptionSupport.FlowTree.Children)
            {
                if (node.Type != NodeType.Finally && node.Type != NodeType.Fault)
                {
                    this.GenerateCodeRecursive(node, instructionBlocks);
                }
            }
            if (this._methodReference.ReturnType.IsNotVoid())
            {
                Instruction instruction = this._methodDefinition.Body.Instructions.LastOrDefault<Instruction>();
                if (instruction != null && instruction.OpCode != OpCodes.Ret && instruction.OpCode != OpCodes.Throw && instruction.OpCode != OpCodes.Rethrow && !(instruction.Operand is Instruction))
                {
                    this._writer.WriteLine("il2cpp_codegen_no_return();");
                }
            }
            this._variableSizedTypeSupport.GenerateInitializerStatements(this._context);
        }
    }
}

=============================================

using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;


public class HybridCLRConfig
{
    public static HybridCLRConfig Ins { get; } = new HybridCLRConfig();

    private readonly HashSet<string> _dheAssemblyNames = new HashSet<string>();

    HybridCLRConfig()
    {
        string assembliesStr = Environment.GetEnvironmentVariable("DHE_ASSEMBLIES");
        if (!string.IsNullOrWhiteSpace(assembliesStr))
        {
            foreach (var ass in assembliesStr.Split(","))
            {
                _dheAssemblyNames.Add(ass + ".dll");
            }
        }
    }

    public bool IsDifferentialHybridAssembly(string assName)
    {
        return _dheAssemblyNames.Contains(assName);
    }
}


===


namespace Unity.IL2CPP.CodeWriters
{
    // Token: 0x0200045A RID: 1114
    public static partial class CodeWriterExtensions
    {
        // Token: 0x06001B16 RID: 6934 RVA: 0x0006F5C8 File Offset: 0x0006D7C8
        public static void WriteMethodWithMetadataInitialization(this IGeneratedMethodCodeWriter writer, string methodSignature, Action<IGeneratedMethodCodeWriter, IRuntimeMetadataAccess> writeMethodBody, string uniqueIdentifier, MethodReference methodRef, bool writingMethodBody = false)
        {
            string identifier = uniqueIdentifier + "_MetadataUsageId";
            MethodMetadataUsage methodMetadataUsage = new MethodMetadataUsage();
            MethodUsage methodUsage = new MethodUsage();
            if (methodRef != null && !methodRef.IsSharedMethod(writer.Context) && HybridCLRConfig.Ins.IsDifferentialHybridAssembly(methodRef.Module.Name))
            {
                methodMetadataUsage.AddInflatedMethod(methodRef, false);
            }
