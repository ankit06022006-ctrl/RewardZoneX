# RewardZoneX
 import { Controller, Post, Body, ForbiddenException, UnauthorizedException } from '@nestjs/common';
import * as admin from 'firebase-admin';
import { PrismaService } from '../../database/prisma.service';
import { LedgerService } from '../wallet/ledger.service';
import { AccountType, Role } from '@prisma/client';
import { Decimal } from '@prisma/client/runtime/library';

@Controller('api/v1/auth')
export class GoogleAuthController {
  constructor(
    private readonly prisma: PrismaService,
    private readonly ledger: LedgerService,
  ) {}

  @Post('google-authenticate')
  async authenticateGoogleUser(
    @Body() body: { idToken: string; email: string; displayName: string; avatarUrl?: string; deviceFingerprint: string }
  ) {
    if (!body.deviceFingerprint) {
      throw new ForbiddenException('Security Alert: Device fingerprint missing');
    }

    // 1. Firebase Token Verification
    let decodedToken;
    try {
      decodedToken = await admin.auth().verifyIdToken(body.idToken);
    } catch (e) {
      throw new UnauthorizedException('Invalid Google Authentication Token');
    }

    const { email, uid } = decodedToken;
    const userEmail = body.email || email;

    return await this.prisma.$transaction(async (tx) => {
      // 2. Check if user already exists with this Google Email
      const existingUser = await tx.user.findUnique({
        where: { email: userEmail },
      });

      if (existingUser) {
        // Purana User Login Flow
        // Check: Kya yeh user wahi hai jiska yeh phone hai?
        if (existingUser.deviceFingerprint && existingUser.deviceFingerprint !== body.deviceFingerprint) {
          // Device mismatch: User ne doosre phone me login karne ki koshish ki
          // Production standard: Ya to OTP verification maango ya update allow karo
          await tx.user.update({
            where: { id: existingUser.id },
            data: { deviceFingerprint: body.deviceFingerprint },
          });
        }

        return {
          success: true,
          isNewUser: false,
          user: {
            id: existingUser.id,
            email: existingUser.email,
            displayName: existingUser.displayName,
          },
        };
      }

      // 3. New User Registration Flow (Yahan 1-Device-1-Account check hoga)
      const existingDeviceAccount = await tx.user.findFirst({
        where: { deviceFingerprint: body.deviceFingerprint },
      });

      if (existingDeviceAccount) {
        // Fraud Alert: Iss phone me pehle se kisi aur ka account register hai!
        throw new ForbiddenException(
          'Security Alert: Iss phone par pehle se ek RewardZone X account register hai. Ek phone me doosra account banana mana hai.'
        );
      }

      // 4. Sab theek hai: Create New User & Lock Device
      const newUser = await tx.user.create({
        data: {
          email: userEmail,
          handle: `user_${uid.substring(0, 8)}`,
          displayName: body.displayName || 'New Earner',
          avatarUrl: body.avatarUrl,
          deviceFingerprint: body.deviceFingerprint, // Device lock ho gaya
          role: Role.USER,
        },
      });

      // 5. Create Double-Entry Ledger Accounts (Zero inflation)
      const userWallet = await tx.ledgerAccount.create({
        data: {
          userId: newUser.id,
          type: AccountType.LIABILITY_USER_BALANCE,
          balanceCache: new Decimal('0.0000'),
        },
      });

      await tx.ledgerAccount.create({
        data: {
          userId: newUser.id,
          type: AccountType.LIABILITY_CREATOR_BALANCE,
          balanceCache: new Decimal('0.0000'),
        },
      });

      // 6. Give 50 Welcome Coins (₹0.50) from Promotional Reserve
      const systemExpense = await tx.ledgerAccount.findFirstOrThrow({
        where: { type: AccountType.EXPENSE_PROMOTION_REWARD },
      });

      await this.ledger.executeTransfer({
        idempotencyKey: `SIGNUP_REWARD_${newUser.id}`,
        sourceAccountId: systemExpense.id,
        destinationAccountId: userWallet.id,
        amount: new Decimal('50.0000'),
        description: 'New User Registration Welcome Bonus',
      });

      return {
        success: true,
        isNewUser: true,
        user: {
          id: newUser.id,
          email: newUser.email,
          displayName: newUser.displayName,
        },
      };
    });
  }
}